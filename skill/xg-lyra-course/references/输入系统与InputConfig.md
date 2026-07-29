# Input System & InputConfig

## Overview

Lyra's input system is based on **Enhanced Input**, decoupling input actions from GameplayAbilities via **Tag mapping**. Core idea: input generates a GameplayTag, and the Tag drives ability activation.

---

## InputConfig Data Structure

**File**: `Source/LyraGame/Input/LyraInputConfig.h`

```cpp
USTRUCT(BlueprintType)
struct FLyraInputAction
{
    GENERATED_BODY()

    // GameplayTag corresponding to the input, e.g., InputTag.Move, InputTag.Jump
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    FGameplayTag InputTag;

    // Corresponding Enhanced Input Action asset
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TObjectPtr<UInputAction> InputAction = nullptr;
};

UCLASS(BlueprintType)
class ULyraInputConfig : public UDataAsset
{
    GENERATED_BODY()

public:
    // All Tag→InputAction mappings
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TArray<FLyraInputAction> InputActions;

    // Find the corresponding InputAction by Tag
    UFUNCTION(BlueprintCallable)
    const UInputAction* FindInputActionForTag(const FGameplayTag& Tag) const
    {
        for (const FLyraInputAction& Action : InputActions)
        {
            if (Action.InputTag == Tag)
            {
                return Action.InputAction;
            }
        }
        return nullptr;
    }
};
```

---

## Binding Flow

### Step 1: Set Up Input Mapping Context

**File**: `Source/LyraGame/Character/LyraHeroComponent.h`

```cpp
void ULyraHeroComponent::InitializePlayerInput(UInputComponent* PlayerInputComponent)
{
    APlayerController* PC = GetController<APlayerController>();
    ULocalPlayer* LP = Cast<ULocalPlayer>(PC->GetLocalPlayer());
    UEnhancedInputLocalPlayerSubsystem* Subsystem = LP->GetSubsystem<UEnhancedInputLocalPlayerSubsystem>();

    // Add input mapping context
    // LyraInputMapping is a UInputMappingContext, defined in Content
    Subsystem->AddMappingContext(LyraInputMapping, Priority);

    // Bind ability activation
    ULyraInputComponent* LyraInputComp = ...;
    LyraInputComp->BindAbilityActions(InputConfig, InputHandles,
        this, &ThisClass::OnInputStarted, &ThisClass::OnInputTriggered, &ThisClass::OnInputCompleted);
}
```

### Step 2: Tag Binding to GA

**File**: `Source/LyraGame/Input/LyraInputComponent.h`

```cpp
void ULyraInputComponent::BindAbilityActions(
    ULyraInputConfig* InputConfig,
    TArray<uint32>& BindHandles,
    UObject* Object,
    FInputActionHandler StartedHandler,
    FInputActionHandler TriggeredHandler,
    FInputActionHandler CompletedHandler)
{
    for (const FLyraInputAction& Action : InputConfig->InputActions)
    {
        if (Action.InputAction)
        {
            // Bind three-stage callbacks for each InputAction
            uint32 Handle = BindNativeAction(
                Action.InputAction,
                ETriggerEvent::Started,
                Object,
                StartedHandler,
                Action.InputTag
            );
            BindHandles.Add(Handle);

            Handle = BindNativeAction(
                Action.InputAction,
                ETriggerEvent::Triggered,
                Object,
                TriggeredHandler,
                Action.InputTag
            );
            BindHandles.Add(Handle);

            Handle = BindNativeAction(
                Action.InputAction,
                ETriggerEvent::Completed,
                Object,
                CompletedHandler,
                Action.InputTag
            );
            BindHandles.Add(Handle);
        }
    }
}
```

### Step 3: HeroComponent Handles Input Events

```cpp
void ULyraHeroComponent::OnInputStarted(const FInputActionInstance& InputAction)
{
    // Get the bound Tag
    FGameplayTag InputTag = InputAction.GetSourceObject()->GetGameplayTag();

    // Activate the corresponding GA through AbilitySystemComponent
    if (ASC && ASC->IsOwnerActorAuthoritative())
    {
        // Direct input activation
        ASC->AbilityLocalInputPressed(InputTag);
    }
}
```

---

## Input Modifiers

Lyra uses Enhanced Input's Modifier mechanism for input preprocessing:

| Modifier | Function |
|----------|----------|
| `UInputModifierDeadZone` | Joystick dead zone processing |
| `UInputModifierNegate` | Invert input values |
| `UInputModifierScalar` | Scale input values |
| `UInputModifierFOVScaling` | Scale mouse sensitivity based on FOV |

Mouse sensitivity scaling is a Lyra-specific approach:
```
MouseSensitivity = BaseSensitivity * FOVScalingFactor
```
Where `FOVScalingFactor` is calculated by `UInputModifierFOVScaling` based on the current CameraMode's FOV.

---

## Complete Chain

```
Player presses button/moves joystick
    → EnhancedInput triggers UInputAction
    → Input Modifier processing (dead zone/scale/FOV)
    → ULyraInputComponent bound callbacks
    → HeroComponent::OnInputStarted/OnInputTriggered/OnInputCompleted
    → Tag matching (InputTag.Move → ASC activates GA)
    → Corresponding GameplayAbility executes
```

---

## Adding New Input Bindings

```cpp
// 1. Create UInputAction and UInputMappingContext in Content
// 2. Add Tag→InputAction mapping in the InputConfig data asset

// 3. Set InputTag in GA
UCLASS()
class UMyShootAbility : public ULyraGameplayAbility
{
    GENERATED_BODY()

public:
    UMyShootAbility()
    {
        // Must match the Tag configured in InputConfig
        AbilityTags.AddTag(FGameplayTag::RequestGameplayTag("InputTag.Shoot"));
    }
};
```

---

## Key Design Points

1. **Tag-driven** — Input and GA are connected via FGameplayTag, neither needs a direct reference to the other
2. **Three-stage callbacks** — Started/Triggered/Completed correspond to press, hold, and release respectively
3. **Mapping context priority** — Multiple input contexts control override relationships through priority (e.g., game input blocked when UI is open)
4. **Local player subsystem** — EnhancedInputLocalPlayerSubsystem manages each player's input mapping contexts
