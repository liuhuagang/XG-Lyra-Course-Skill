# GAS Architecture Overview

> Corresponding lecture: [080_GAS Architecture](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/docs/UE5_Lyra学习指南_080_GAS的架构.md)

## ULyraAbilitySet

[ULyraAbilitySet](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/LyraAbilitySet.h) is a data asset inheriting from `UPrimaryDataAsset`, used to package a set of GAS resources and grant them to an ASC.

**Core Data Structures:**

| Struct | Purpose | Fields |
|--------|------|------|
| `FLyraAbilitySet_GameplayAbility` | GameplayAbility to grant | `Ability` (TSubclassOf\<ULyraGameplayAbility\>), `AbilityLevel` (int32), `InputTag` (FGameplayTag) |
| `FLyraAbilitySet_GameplayEffect` | GameplayEffect to grant | `GameplayEffect` (TSubclassOf\<UGameplayEffect\>), `EffectLevel` (float) |
| `FLyraAbilitySet_AttributeSet` | AttributeSet to grant | `AttributeSet` (TSubclassOf\<UAttributeSet\>) |
| `FLyraAbilitySet_GrantedHandles` | Collection of granted handles | `AbilitySpecHandles`, `GameplayEffectHandles`, `GrantedAttributeSets` |

**Core Methods:**
- `GiveToAbilitySystem(ULyraAbilitySystemComponent*, FLyraAbilitySet_GrantedHandles*, UObject* OverrideSourceObject = nullptr)` — Grants the AbilitySet to the specified ASC
- `TakeFromAbilitySystem(ULyraAbilitySystemComponent*)` — Revokes all granted content from the ASC

**Member Variables:**
- `GrantedGameplayAbilities` — Array defining GAs to grant
- `GrantedGameplayEffects` — Array defining GE to grant
- `GrantedAttributes` — Array defining AttributeSets to grant

InputTag is stored in `AbilitySpec.GetDynamicSpecSourceTags().AddTag(InputTag)`, and the input system later looks up the corresponding AbilitySpec through this Tag.

## ILyraAbilitySourceInterface

Ability source interface, defining damage attenuation calculations. Implemented by objects such as weapons.

**Core Methods:**
- `GetDistanceAttenuation(float Distance, const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags) const` — Distance attenuation
- `GetPhysicalMaterialAttenuation(const UPhysicalMaterial* PhysicalMaterial, const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags) const` — Physical material attenuation

Used in GEEC via FLyraGameplayEffectContext to compute damage attenuation.

## ULyraAbilitySystemComponent

[ULyraAbilitySystemComponent](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.h) is Lyra's core extension of UAbilitySystemComponent, responsible for input routing, activation group management, tag relationship mapping, dynamic tag GE, and more. See [02-ASC与输入激活.md]() for details.

**Extended Key Features:**
- Input tag-driven ability activation pipeline
- Activation groups (Independent / Exclusive_Replaceable / Exclusive_Blocking)
- Tag relationship mapping (Block / Cancel / Required / Blocked Tags)
- Dynamic tag GameplayEffect management
- Global AbilitySystem registration

## ULyraGameplayAbility

[ULyraGameplayAbility](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/Abilities/LyraGameplayAbility.h) is the base class for all Lyra GameplayAbilities.

**Core Extensions:**
- `ActivationPolicy` — Activation policy: `OnInputTriggered` (trigger on press), `WhileInputActive` (sustained while held), `OnSpawn` (activate on spawn)
- `ActivationGroup` — Activation group: `Independent`, `Exclusive_Replaceable`, `Exclusive_Blocking`
- `AdditionalCosts` — Additional costs (TagStack, InventoryItem, etc.)
- `ActiveCameraMode` — Camera mode to switch to when active
- `OnAbilityFailedToActivate(FGameplayTagContainer)` — Activation failure callback, broadcast via UGameplayMessageSubsystem

## ULyraAbilitySystemGlobals

[ULyraAbilitySystemGlobals](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/LyraAbilitySystemGlobals.h) configuration class, marked `Config=Game`.

**Overridden Methods:**
- `AllocGameplayEffectContext()` — Returns `FLyraGameplayEffectContext` instance, replacing the default FGameplayEffectContext

Configured in `DefaultGame.ini`:

```ini
[/Script/GameplayAbilities.AbilitySystemGlobals]
GlobalGameplayCueManagerClass=/Script/LyraGame.LyraGameplayCueManager
+GameplayCueNotifyPaths=/Script/LyraGame
ReplicateActivationOwnedTags=True
MinimalReplicationTagCountISM=16
AbilitySystemInvalidationServerKey=1
AbilitySystemInvalidationClientKey=2
ActivateFailCooldownTag=/Script/LyraGame.Default.__GameplayAbilityActivateFailCooldown
ActivateFailCostTag=/Script/LyraGame.Default.__GameplayAbilityActivateFailCost
ActivateFailNetworkingTag=/Script/LyraGame.Default.__GameplayAbilityActivateFailNetworking
ActivateFailTagsBlockedTag=/Script/LyraGame.Default.__GameplayAbilityActivateFailTagsBlocked
ActivateFailTagsMissingTag=/Script/LyraGame.Default.__GameplayAbilityActivateFailTagsMissing
bUseDebugTargetFromHud=True
```

## ULyraAbilityTagRelationshipMapping

[ULyraAbilityTagRelationshipMapping](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/LyraAbilityTagRelationshipMapping.h) is a DataAsset defining relationships between AbilityTags.

**FLyraAbilityTagRelationship Struct:**
- `AbilityTag` — Ability tag
- `AbilityTagsToBlock` — Tags of other abilities blocked when this ability is active
- `AbilityTagsToCancel` — Tags of other abilities canceled when this ability is active
- `ActivationRequiredTags` — Additional required activation tags
- `ActivationBlockedTags` — Additional blocked activation tags

**Test Helper Method:** `IsAbilityCancelledByTag()` — Checks whether an ability is canceled by a given action tag.

## FLyraGameplayEffectContext

[FLyraGameplayEffectContext](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/LyraGameplayEffectContext.h) extends `FGameplayEffectContext` with network serialization support.

**Extended Fields:**
- `CartridgeID` (int32) — Bullet batch ID
- `AbilitySourceObject` (TWeakObjectPtr\<const UObject\>) — Ability source object (not network synced)

**Core Methods:**
- `ExtractEffectContext(FGameplayEffectContextHandle)` — Static method to extract this type from a Handle
- `SetAbilitySource(const ILyraAbilitySourceInterface*, float)` — Sets source and level
- `GetAbilitySource()` — Gets source interface
- `Duplicate()` — Deep copy (including HitResult)

## FLyraGameplayAbilityTargetData_SingleTargetHit

Extends `FGameplayAbilityTargetData_SingleTargetHit`, adds `CartridgeID` field, synchronized during network serialization. `AddTargetDataToContext()` propagates CartridgeID from TargetData to EffectContext.

## ULyraGlobalAbilitySystem

[ULyraGlobalAbilitySystem](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/LyraGlobalAbilitySystem.h) is a `UWorldSubsystem` that manages globally registered ASCs, supporting batch application/removal of abilities and effects.

**Helper Structures:**
- `FGlobalAppliedAbilityList` — `Handles` (Map\<ASC, SpecHandle\>), methods: `AddToASC`, `RemoveFromASC`, `RemoveFromAll`
- `FGlobalAppliedEffectList` — Same, but manages EffectHandle

**Core Methods:**
- `ApplyAbilityToAll(TSubclassOf\<UGameplayAbility\>)` — Applies ability to all registered ASCs
- `ApplyEffectToAll(TSubclassOf\<UGameplayEffect\>)` — Applies effect to all registered ASCs
- `RemoveAbilityFromAll` / `RemoveEffectFromAll`
- `RegisterASC(ULyraAbilitySystemComponent*)` — Registers ASC and applies current global effects
- `UnregisterASC(ULyraAbilitySystemComponent*)` — Unregisters and removes global effects

**Registration Timing:** Called in `ULyraAbilitySystemComponent::InitAbilityActorInfo()`: `GlobalAbilitySystem->RegisterASC(this)`; `EndPlay()` calls `UnregisterASC(this)`.

## ALyraTaggedActor

An Actor implementing the `IGameplayTagAssetInterface`, configurable with GameplayTags in editor mode. Used in the avatar system to mark tags like gender/body type.

## Damage/Heal Calculation

- **ULyraDamageExecution** (`UGameplayEffectExecutionCalculation`) — Damage execution calculation, captures BaseDamage, computes final damage combining attenuation and team detection
- **ULyraHealExecution** — Heal execution calculation, captures BaseHeal, outputs Healing attribute

See [06-GEEC伤害计算.md]() for details.

## ULyraAbilityCost

[ULyraAbilityCost](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/Abilities/LyraAbilityCost.h) is an object base class (`EditInlineNew`) defining the ability cost interface.

**Methods:**
- `CheckCost()` — Checks whether the cost can be paid
- `ApplyCost()` — Applies the cost
- `ShouldOnlyApplyCostOnHit()` — Whether the cost should only be applied on hit

**Derived Classes:**
- `LyraAbilityCost_PlayerTagStack` — Consumes player TagStack
- `LyraAbilityCost_ItemTagStack` — Consumes item TagStack
- `LyraAbilityCost_InventoryItem` — Consumes inventory item
