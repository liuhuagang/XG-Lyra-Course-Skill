# ASC & Input Activation

> Corresponding lecture: [081_ASC与GA](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/docs/UE5_Lyra学习指南_081_ASC与GA.md)
> Core code: [LyraAbilitySystemComponent](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.h)

## ULyraAbilitySystemComponent

ULyraAbilitySystemComponent is an extension of UAbilitySystemComponent that manages Lyra's unique ability activation pipeline.

## Ability Grant Flow

```
PawnData
  └── AbilitySets[] (TArray<TObjectPtr<ULyraAbilitySet>>)
        └── ALyraPlayerState::SetPawnData()
              └── Iterates AbilitySets, calls GiveToAbilitySystem()
                    └── Registers Abilities / Effects / AttributeSets
                          └── Sends NAME_LyraAbilityReady frame event
```

## Input Activation Pipeline

```
Enhanced Input Action
  └── ULyraHeroComponent::Input_AbilityInputTagPressed(InputTag)
        └── ULyraAbilitySystemComponent::AbilityInputTagPressed(InputTag)
              ├── Iterates ActivatableAbilities, matches InputTag
              ├── Buckets by ActivationPolicy:
              │     ├── OnInputTriggered → InputPressedSpecHandles
              │     └── WhileInputActive → InputHeldSpecHandles
              │
              └── ALyraPlayerController::PostProcessInput(DeltaTime)
                    └── ULyraAbilitySystemComponent::ProcessAbilityInput(DeltaTime, bGamePaused)
                          ├── Phase 1: Process held input (WhileInputActive)
                          │     └── Try to activate abilities in InputHeldSpecHandles still in Pressed list
                          │
                          ├── Phase 2: Process press-triggered (OnInputTriggered)
                          │     └── Try to activate abilities in InputPressedSpecHandles
                          │
                          └── Phase 3: Clear
                                └── ClearAbilityInput() → Resets InputPressedSpecHandles & InputReleasedSpecHandles
```

**Input Blocking Mechanism:** When the Actor has the `TAG_Gameplay_AbilityInputBlocked` tag, `ProcessAbilityInput()` skips all processing.

### AbilitySpecInputPressed/Released

Lyra does not directly use `UGameplayAbility::bReplicateInputDirectly`. Instead, it triggers `WaitInputPress` / `WaitInputRelease` AbilityTask replication events through the ASC's `InvokeReplicatedEvent()`. This is detailed in [03-GA_Jump与GA_Dash.md]().

## ActorInfo Initialization Locations

| Scheme | Owner | Avatar | Applicable Scenario |
|------|-------|--------|----------|
| PlayerState | PlayerState | Pawn | Standard player (prevents loss on Pawn respawn) |
| GameState | GameState | GameState | GameMode-level global abilities |
| PawnExtensionComponent | InOwnerActor | Pawn | Dynamically set via GameFeature |
| LyraCharacterWithAbilities | Self | Self | Simple non-player characters |

`InitAbilityActorInfo()` also includes the following operations:
- `TryActivateAbilitiesOnSpawn()` — Attempts to activate OnSpawn strategy abilities
- `ULyraGlobalAbilitySystem::RegisterASC(this)` — Registers to the global system (see [01-GAS架构概述.md]())

## TryActivateAbilitiesOnSpawn

Called in `InitAbilityActorInfo()`, iterates `ActivatableAbilities`, checks the `ULyraGameplayAbility` CDO's `TryActivateAbilityOnSpawn()` return value, and attempts activation if true.

Also called in `OnGiveAbility()` to ensure dynamically registered abilities can be activated on spawn.

## Activation Group Management

`ELyraAbilityActivationGroup` enum:

| Enum Value | Behavior |
|--------|------|
| `Independent` | No mutual interference |
| `Exclusive_Replaceable` | When activated, can be replaced by other Exclusive abilities |
| `Exclusive_Blocking` | When activated, blocks other Exclusive abilities |

Maintains current counts per group via `ActivationGroupCounts[]` array:

- `IsActivationGroupBlocked(Group)` — Determines if blocked based on count
- `AddAbilityToActivationGroup(Group, Handle)` — Increments count; calls `ChangeActivationGroup` to switch if Handle exists
- `RemoveAbilityFromActivationGroup(Group, Handle)` — Decrements count
- `CancelActivationGroupAbilities(Group, ...)` — Cancels abilities in the specified group (via `CancelAbilitiesByFunc`)

### Ability Group Switching Flow

```
ULyraGameplayAbility::CanChangeActivationGroup(NewGroup)
  └── ASC::IsActivationGroupBlocked(NewGroup)
        └── If blocked, reject the switch

ULyraGameplayAbility::ChangeActivationGroup(NewGroup)
  ├── CanChangeActivationGroup(NewGroup) → Check
  ├── RemoveAbilityFromActivationGroup(OldGroup)
  ├── Update ActivationGroup
  └── AddAbilityToActivationGroup(NewGroup)
        └── If NewGroup is Exclusive, automatically cancel other abilities in the same group
```

```cpp
void ULyraAbilitySystemComponent::AddAbilityToActivationGroup(ELyraAbilityActivationGroup Group, FGameplayAbilitySpecHandle Handle)
{
    ++ActivationGroupCounts[Group];
    if (Handle.IsValid())
    {
        // If the ability is not yet counted in the activation group, call ChangeActivationGroup to adjust
    }
}
```

## Activation Check & Tag Relationships

`DoesAbilitySatisfyTagRequirements()` overrides the UGameplayAbility version:

```
Perform standard Tag check
  └── Via ASC::GetAdditionalActivationTagRequirements(Tag, OutRequired, OutBlocked)
        └── Delegates to ULyraAbilityTagRelationshipMapping::GetRequiredAndBlockedActivationTags()
              └── Queries FLyraAbilityTagRelationship
                    ├── ActivationRequiredTags → Added to AllRequiredTags
                    └── ActivationBlockedTags → Added to AllBlockedTags

Check AllRequiredTags and AllBlockedTags
```

TagRelationshipMapping is set via `SetTagRelationshipMapping()`, typically configured in PawnData.

## Ability Failure Notification

```
ASC::NotifyAbilityFailed(ActorInfo, Handle, FailureTag)
  └── ClientNotifyAbilityFailed (RPC to client)
        └── ULyraAbilitySystemComponent::HandleAbilityFailed(...)
              └── ULyraGameplayAbility::OnAbilityFailedToActivate(FailureTagContainer)
                    └── NativeOnAbilityFailedToActivate(FailureTag)
                          └── Broadcast via UGameplayMessageSubsystem
                                ├── FailureTagToUserFacingMessages → Display user message
                                └── FailureTagToAnimMontage → Play failure animation
```

## Dynamic Tag GE

```cpp
void ULyraAbilitySystemComponent::AddDynamicTagGameplayEffect(FGameplayTag Tag)
{
    // Get DynamicTagGameplayEffect from ULyraGameData
    // Create GE Spec, add Tag to DynamicGrantedTags
    // Apply GE
}

void ULyraAbilitySystemComponent::RemoveDynamicTagGameplayEffect(FGameplayTag Tag)
{
    // Use FGameplayEffectQuery::MakeQuery_MatchAnyOwningTags({Tag}) to query and remove
}
```

## Ability Camera Mode

In ULyraGameplayAbility, camera modes are switched through `SetCameraMode(TSubclassOf<ULyraCameraMode>)` and `ClearCameraMode()`.

Under the hood, calls `ULyraHeroComponent::SetCameraMode()` / `ClearCameraMode()`, tracking the currently active camera mode via the `ActiveCameraMode` member variable. `ClearCameraMode()` is automatically called in `EndAbility`.

## Ability Costs

In `ULyraGameplayAbility::CheckCost()` and `ApplyCost()`, the base class `UGameplayAbility` version is called first for standard Tag/Mana cost checks, then the `AdditionalCosts` array is iterated for additional costs:

```cpp
bool ULyraGameplayAbility::CheckCost(...)
{
    if (!Super::CheckCost(...)) return false;
    for (ULyraAbilityCost* Cost : AdditionalCosts)
    {
        if (!Cost->CheckCost(this, ...)) return false;
    }
    return true;
}

void ULyraGameplayAbility::ApplyCost(...)
{
    Super::ApplyCost(...);
    for (ULyraAbilityCost* Cost : AdditionalCosts)
    {
        // bOnlyApplyCostOnHit logic: check if hit
        if (Cost->ShouldOnlyApplyCostOnHit())
        {
            // Check if the ability has a hit target
            if (!HasHitTarget()) continue;
        }
        Cost->ApplyCost(this, ...);
    }
}
```
