# GAS Ability System Architecture

## Overview

Lyra extends UE's GAS (GameplayAbilitySystem) with multiple layers, forming a complete **Ability Grant → Activation → Execution → Damage Calculation** system.

---

## AbilitySet: Ability Aggregate

**File**: `Source/LyraGame/AbilitySystem/LyraAbilitySet.h`

AbilitySet is a data asset that aggregates GA, GE, and AttributeSet into a configurable unit.

```cpp
UCLASS(BlueprintType)
class LYRAGAME_API ULyraAbilitySet : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    // GameplayAbilities to grant
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay Abilities")
    TArray<FLyraAbilitySet_GameplayAbility> GrantedGameplayAbilities;

    // GameplayEffects to apply
    UPROPERTY(EditDefaultsOnly, Category = "Gameplay Effects")
    TArray<FLyraAbilitySet_GameplayEffect> GrantedGameplayEffects;

    // AttributeSets to initialize
    UPROPERTY(EditDefaultsOnly, Category = "Attribute Sets")
    TArray<FLyraAbilitySet_AttributeSet> GrantedAttributes;
};

// Grant flow
void ULyraAbilitySet::GiveToAbilitySystem(
    ULyraAbilitySystemComponent* ASC,
    ULyraEquipmentInstance* SourceObject,
    TArray<FGameplayAbilitySpecHandle>& OutHandles) const
{
    // 1. Grant GA
    for (const auto& Entry : GrantedGameplayAbilities)
    {
        FGameplayAbilitySpec Spec(Entry.Ability);
        Spec.SourceObject = SourceObject;
        OutHandles.Add(ASC->GiveAbility(Spec));
    }

    // 2. Apply GE
    for (const auto& Entry : GrantedGameplayEffects)
    {
        FGameplayEffectContextHandle Context = ASC->MakeEffectContext();
        Context.AddSourceObject(SourceObject);
        ASC->BP_ApplyGameplayEffectSpecToSelf(Entry.Effect->MakeSpec());
    }

    // 3. Initialize attribute sets
    for (const auto& Entry : GrantedAttributes)
    {
        ASC->InitStats(Entry.AttributeSetType, nullptr);
    }
}
```

---

## ActivationGroup: Activation Group Management

**File**: `Source/LyraGame/AbilitySystem/Abilities/LyraGameplayAbility.h`

Lyra introduces the `ActivationGroup` concept in `ULyraGameplayAbility` to control behavior when multiple GAs are activated simultaneously:

```cpp
UENUM(BlueprintType)
enum class ELyraAbilityActivationGroup : uint8
{
    // Independent activation, no conflict with other GAs
    Independent,

    // Exclusive replaceable: activating this GA replaces other Exclusive_Replaceable GAs in the same group
    Exclusive_Replaceable,

    // Exclusive blocking: activating this GA blocks other GAs in the same group
    Exclusive_Blocking
};

UCLASS()
class LYRAGAME_API ULyraGameplayAbility : public UGameplayAbility
{
    GENERATED_BODY()

public:
    // This GA's activation group
    UPROPERTY(EditDefaultsOnly, Category = "Lyra|Ability")
    ELyraAbilityActivationGroup ActivationGroup;
};
```

Activation group rules:
- **Independent**: No conflict with any GA, can activate simultaneously (e.g., movement, jump)
- **Exclusive_Replaceable**: Canceled when replaced by Exclusive_Blocking; otherwise can coexist
- **Exclusive_Blocking**: Cancels all Replaceable GAs in the same group on activation

---

## TagRelationshipMapping: Tag Relationship Mapping

**File**: `Source/LyraGame/AbilitySystem/LyraAbilityTagRelationshipMapping.h`

Configures Cancel/Block relationships between GAs via data assets:

```cpp
USTRUCT()
struct FLyraAbilityRelationshipMappingEntry
{
    FGameplayTagContainer AbilityTagsToMatch;
    FGameplayTagContainer CancelAbilitiesWithTag;
    FGameplayTagContainer BlockAbilitiesWithTag;
};

UCLASS()
class ULyraAbilityTagRelationshipMapping : public UDataAsset
{
    GENERATED_BODY()
    // Configuration entries
    UPROPERTY(EditDefaultsOnly)
    TArray<FLyraAbilityRelationshipMappingEntry> AbilityRelationshipMappingEntries;
};
```

The engine framework automatically checks TagRelationshipMapping before activating a GA to decide whether to cancel or block existing GAs.

---

## AttributeSet

Lyra uses a multi-layer AttributeSet design:

| AttributeSet | File Path | Core Attributes |
|--------------|-----------|-----------------|
| ULyraHealthSet | `AbilitySystem/Attributes/LyraHealthSet.h` | Health, MaxHealth, DamageResistance |
| ULyraCombatSet | `AbilitySystem/Attributes/LyraCombatSet.h` | BaseDamage, MoveSpeed |

**ClampAttribute mechanism**: HealthSet uses `FOnAttributeChangeRequest` to intercept attribute changes before they occur, ensuring attribute values stay within bounds:

```
Attribute pre-change event
    → ClampAttribute (checks if exceeding [Min, Max])
    → Corrects to boundary value if out of range
    → Actual attribute change
```

---

## FLyraGameplayEffectContext

**File**: `Source/LyraGame/AbilitySystem/LyraGameplayEffectContext.h`

Extends GE Context to carry additional damage information:

```cpp
USTRUCT()
struct FLyraGameplayEffectContext : public FGameplayEffectContext
{
    // Cartridge ID (used to distinguish damage from different bullets of the same weapon)
    int32 CartridgeID;
};
```

`CartridgeID` is used for damage追溯—when damage is Applied, it can be associated with a specific bullet or weapon instance via CartridgeID.

---

## Damage Calculation (GEEC)

**File**: `Source/LyraGame/AbilitySystem/Executions/LyraDamageExecution.h`

```cpp
UCLASS()
class ULyraDamageExecution : public UGameplayEffectExecutionCalculation
{
    GENERATED_BODY()

public:
    ULyraDamageExecution();

    virtual void Execute_Implementation(
        const FGameplayEffectCustomExecutionParameters& Params,
        FGameplayEffectCustomExecutionOutput& Output) const override;
};
```

Damage formula:
```
FinalDamage = BaseDamage × DistanceAttenuation × PhysicalMaterialAttenuation
              × DamageInteractionAllowedMultiplier
```

| Factor | Source | Description |
|--------|--------|-------------|
| BaseDamage | ULyraCombatSet | Weapon base damage |
| DistanceAttenuation | Curve table | Distance falloff curve |
| PhysicalMaterialAttenuation | Physical material table | Damage reduction by different materials |
| DamageInteractionAllowedMultiplier | DamageInteractionAllowed tag | Whether friendly fire or specific target types are allowed |

---

## ASC Input Activation Pipeline

**File**: `Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.h`

Lyra extends ASC with an explicit **three-stage input processing pipeline**, rather than relying on the default `AbilityLocalInputPressed`:

```cpp
UCLASS()
class LYRAGAME_API ULyraAbilitySystemComponent : public UAbilitySystemComponent
{
    GENERATED_BODY()

public:
    // Input pressed: records to PressedSpecHandles
    void AbilityInputPressed(const FGameplayTag& InputTag);

    // Input released: moves from Pressed to Released
    void AbilityInputReleased(const FGameplayTag& InputTag);

    // Per-frame execution: processes three-stage input
    void ProcessAbilityInput(float DeltaTime, bool bGamePaused);

protected:
    // Inputs pressed this frame
    TArray<FGameplayTag> InputPressedSpecHandles;

    // Inputs held (not released yet)
    TArray<FGameplayTag> InputHeldSpecHandles;

    // Inputs released this frame
    TArray<FGameplayTag> InputReleasedSpecHandles;

    // Whether input is blocked (e.g., when Modal UI is open)
    bool bBlockInput = false;
};
```

### ProcessAbilityInput Three-Stage Processing

```
ProcessAbilityInput executed each frame
    │
    ├── Stage 1: Process Pressed (newly pressed)
    │   → Iterate InputPressedSpecHandles
    │   → Find Tag-matching GA
    │   → Call TryActivateAbility (checks InputBlocked Tag)
    │   → If activation succeeds, add Handle to InputHeldSpecHandles
    │
    ├── Stage 2: Process Held (held continuous)
    │   → Iterate InputHeldSpecHandles
    │   → For sustained GAs (e.g., sprint), check if update is needed
    │
    └── Stage 3: Process Released (released)
        → Iterate InputReleasedSpecHandles
        → Notify corresponding GA that input was released
        → Clean up entries in InputHeldSpecHandles
```

### Input Blocking

When `bBlockInput = true` (e.g., Modal UI layer active), `ProcessAbilityInput` skips all processing but does not clear the queue. Input continues normal processing after the block is lifted.

---

## GA Implementation Template: GA_Jump

Below is a complete GA implementation template (illustrative code, not an actual file in the Lyra repository) showing the standard structure of a GA in Lyra:

```cpp
UCLASS()
class UGA_Jump : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UGA_Jump()
    {
        // Set activation group to Independent, no conflict with other abilities
        ActivationGroup = ELyraAbilityActivationGroup::Independent;

        // Tags owned while active
        ActivationOwnedTags.AddTag(TAG_Gameplay_Ability_Jump);
    }

    // Check if can activate: character must be on the ground
    virtual bool CanActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayTagContainer* SourceTags,
        const FGameplayTagContainer* TargetTags,
        FGameplayTagContainer* OptionalRelevantTags) const override
    {
        if (!Super::CanActivateAbility(Handle, ActorInfo, SourceTags, TargetTags, OptionalRelevantTags))
        {
            return false;
        }

        const ACharacter* Character = Cast<ACharacter>(ActorInfo->AvatarActor);
        return Character && Character->CanJump();
    }

    virtual void ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData) override
    {
        Super::ActivateAbility(Handle, ActorInfo, ActivationInfo, ActorInfo, TriggerEventData);

        // 1. Apply Root Motion (optional)
        // 2. Play Montage
        // 3. Wait for input release (AbilityTask_WaitInputRelease)
        // 4. Apply GameplayEffect (e.g., consume Stamina)
        // 5. CommitAbility consumes Cost

        if (CommitAbility(Handle, ActorInfo, ActivationInfo))
        {
            // Jump successful
            Character->Jump();
        }
        else
        {
            EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
        }
    }

    virtual void EndAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        bool bReplicateEndAbility,
        bool bWasCancelled) override
    {
        // Clean up state
        if (ACharacter* Character = Cast<ACharacter>(ActorInfo->AvatarActor))
        {
            Character->StopJumping();
        }

        Super::EndAbility(Handle, ActorInfo, ActivationInfo, bReplicateEndAbility, bWasCancelled);
    }
};
```

---

## HealthComponent: Health Component & Death State Machine

**File**: `Source/LyraGame/Character/LyraHealthComponent.h`

`ULyraHealthComponent` is the bridge between attribute sets (data layer) and game events (presentation layer):

```cpp
UCLASS()
class ULyraHealthComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    // Bind to ASC's HealthSet
    void InitializeWithAbilitySystem(ULyraAbilitySystemComponent* ASC);

    // Unbind
    void UninitializeFromAbilitySystem();

    // Death state query
    bool IsDeadOrDying() const;
    ELyraDeathState GetDeathState() const;

    // Trigger death
    void StartDeath();
    void FinishDeath();

    // Event delegates
    FOnHealthChangedDelegate OnHealthChanged;
    FOnHealthChangedDelegate OnMaxHealthChanged;
    FOnDeathStartedDelegate OnDeathStarted;
    FOnDeathFinishedDelegate OnDeathFinished;

protected:
    // Three-state death state machine
    ELyraDeathState DeathState;
};
```

### Death State Machine

```
NotDead ──→ DeathStarted ──→ DeathFinished
               │                    │
               │ Disable input      │ Clear GameplayTag
               │ Play death montage │ Disable collision
               │ Unpossess Pawn     │ Destroy/hide Actor
               ▼                    ▼
           OnDeathStarted      OnDeathFinished
```

### Call Flow

```
GE applies damage
    → ULyraDamageExecution (GEEC) calculates final damage
    → HealthSet::Health attribute decreases
    → HealthComponent::OnHealthChanged callback
    → Detects Health <= 0
    → StartDeath()
        → Broadcasts OnDeathStarted
        → Disables input
        → Plays death animation
    → FinishDeath()
        → Broadcasts OnDeathFinished
        → Clears GameplayTag
        → Hides/destroys character
```

---

## AbilityCost: Cost System

**File**: `Source/LyraGame/AbilitySystem/Abilities/LyraAbilityCost.h`

Lyra implements extensible ability costs through the `ULyraAbilityCost` abstract class:

```cpp
UCLASS(Abstract, DefaultToInstanced, EditInlineNew)
class ULyraAbilityCost : public UObject
{
    GENERATED_BODY()

public:
    // Execute cost check (can pay)
    virtual bool CheckCost(
        const ULyraGameplayAbility* Ability,
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        FGameplayTagContainer* OptionalRelevantTags) const;

    // Execute cost deduction
    virtual void ApplyCost(
        const ULyraGameplayAbility* Ability,
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo);
};
```

### Built-in Cost Types

| Cost Class | Usage |
|------------|-------|
| ULyraAbilityCost_ItemTagStack | Consumes TagStack count on InventoryItemInstance (e.g., ammo) |
| ULyraAbilityCost_AttributeSetBased | AttributeSet-based cost (e.g., stamina) |

Configuration: Add to `AdditionalCosts` array in `ULyraGameplayAbility`:
```cpp
UCLASS()
class UMyShootAbility : public ULyraGameplayAbility
{
    // Configure AdditionalCosts in the editor
    // Example: ULyraAbilityCost_ItemTagStack → consumes 1 ammo per shot
};
```

---

## Key Design Points

1. **AbilitySet is swappable** — GA/GE/AttributeSet are combined as data assets, grantable and revocable at runtime through EquipmentInstance
2. **ActivationGroup as strategy** — GA concurrency control is elevated from hardcoded to declarative configuration
3. **TagRelationshipMapping externalized** — Cancel/Block relationships are placed in data assets, no C++ code changes needed to adjust ability relationships
4. **GE Context extensible** — CartridgeID mechanism enables precise attribution for multi-damage per shot
5. **ProcessAbilityInput three-stage pipeline** — Pressed/Held/Released are bucketed separately, supporting input blocking and sustained abilities
6. **GA template standard structure** — CanActivateAbility → ActivateAbility → CommitAbility → Execution → EndAbility
7. **HealthComponent death state machine** — NotDead→DeathStarted→DeathFinished three-state transition, delegate-driven event broadcasting
