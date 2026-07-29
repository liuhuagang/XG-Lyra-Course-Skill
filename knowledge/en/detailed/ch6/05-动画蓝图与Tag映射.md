# Animation Blueprint & Tag Mapping

## ULyraAnimInstance

`ULyraAnimInstance` inherits from `UAnimInstance` and is the base class for all animation blueprints in Lyra.

### Ground Distance

```cpp
UPROPERTY(BlueprintReadOnly)
float GroundDistance;
```

- Retrieved from `ULyraCharacterMovementComponent::GetGroundInfo()` in `NativeUpdateAnimation`
- Used for blend spaces or state machine control in animation blueprints

### Initialization

```cpp
void ULyraAnimInstance::NativeInitializeAnimation()
{
    Super::NativeInitializeAnimation();
    AActor* OwningActor = GetOwningActor();
    if (UAbilitySystemComponent* ASC =
        UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(OwningActor))
    {
        GameplayTagPropertyMap.Initialize(this, ASC);
    }
}
```

- Gets ASC via `UAbilitySystemGlobals::GetAbilitySystemComponentFromActor`
- Registers and binds GameplayTagPropertyMap with ASC

## FGameplayTagBlueprintPropertyMap

Automatically maps GameplayTag states to animation blueprint property variables.

### Data Structure

```cpp
struct FGameplayTagBlueprintPropertyMapping {
    FGameplayTag TagToMap;                           // Tag to listen for
    TFieldPath<FProperty> PropertyToEdit;            // Property reference to set
    FName PropertyName;                              // Property name (for lookup)
    FGuid PropertyGuid;                              // Property GUID
    FDelegateHandle DelegateHandle;                  // Event handle
};

struct FGameplayTagBlueprintPropertyMap {
    TArray<FGameplayTagBlueprintPropertyMapping> PropertyMappings;
    TWeakObjectPtr<UObject> CachedOwner;              // Cached Owner
    TWeakObjectPtr<UAbilitySystemComponent> CachedASC; // Cached ASC
};
```

### Registration Flow

```cpp
void Initialize(UObject* Owner, UAbilitySystemComponent* ASC)
{
    CachedOwner = Owner;
    CachedASC = ASC;
    for (auto& Mapping : PropertyMappings)
    {
        // Resolve property TFieldPath
        // Check property type (bool/int/float)
        // Register GameplayTag event
        ASC->RegisterAndCallGameplayTagEvent(
            Mapping.TagToMap,
            FOnGameplayTagValueChange::FDelegate::CreateUObject(
                Owner, &UGameplayTagBlueprintPropertyMap::GameplayTagEventCallback,
                &Mapping),
            EGameplayTagEventType::NewOrRemoved);
    }
}
```

### Callback Handling

```cpp
void GameplayTagEventCallback(
    const FGameplayTag Tag, int32 NewCount,
    FGameplayTagBlueprintPropertyMapping* RegisteredMapping)
{
    // Find the corresponding PropertyMapping by Tag
    // Set value based on property type:
    switch (PropertyType) {
        case FBoolProperty:  *bool_value   = NewCount > 0;     break;
        case FIntProperty:   *int32_value  = NewCount;          break;
        case FFloatProperty: *float_value  = float(NewCount);   break;
    }
}
```

- bool: `NewCount > 0` = true, otherwise false
- int: Directly assigns `NewCount`
- float: Implicit conversion `float(NewCount)`

## Typical Usage

Expose `GameplayTagPropertyMap` as an `EditDefaultsOnly` editable property in animation blueprints, configured in the details panel:

| Tag | Property Name | Type | Effect |
|-----|--------|------|------|
| `Status.Crouching` | bIsCrouching | bool | Blend to crouch animation when crouching |
| `Movement.Mode.Walking` | bIsWalking | bool | Blend to walk animation when walking |
| `Status.Stunned` | StunCount | int | Play hit reaction animation when stunned |

## Code References

| Class/Struct | File |
|---------|------|
| `ULyraAnimInstance` | [LyraAnimInstance.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Animation/LyraAnimInstance.h) |
| `FGameplayTagBlueprintPropertyMap` | [LyraAnimInstance.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Animation/LyraAnimInstance.h) |
| `FGameplayTagBlueprintPropertyMapping` | [LyraAnimInstance.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Animation/LyraAnimInstance.h) |
