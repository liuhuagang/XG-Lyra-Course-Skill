# Attribute Sets & HealthSet

> Corresponding lecture: [085_讲解属性集](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/docs/UE5_Lyra学习指南_085_讲解属性集.md)
> Core code: [LyraAttributeSet](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/Attributes/), [LyraHealthSet](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/Attributes/LyraHealthSet.h)

## Attribute Set Hierarchy

```
UAttributeSet (Engine base class)
  └── ULyraAttributeSet (Lyra base class, adds initialization check)
        ├── ULyraCombatSet (Combat attributes: BaseDamage, BaseHeal)
        └── ULyraHealthSet (Health attributes: Health, MaxHealth, Healing, Damage)
```

## ULyraCombatSet

[ULyraCombatSet](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/Attributes/LyraCombatSet.h) defines basic combat attributes:

| Attribute | Type | Description |
|------|------|------|
| `BaseDamage` | FGameplayAttributeData | Base damage value |
| `BaseHeal` | FGameplayAttributeData | Base heal value |

Uses the `ATTRIBUTE_ACCESSORS(ULyraCombatSet, BaseDamage)` macro to generate `GetBaseDamage()`, `SetBaseDamage()`, `InitBaseDamage()` methods.

## ULyraHealthSet

[ULyraHealthSet](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/Attributes/LyraHealthSet.h) is the core attribute set for the health system.

### Attribute Definitions

| Attribute | Replicated | Description |
|------|-----------|------|
| `Health` | Yes | Current health |
| `MaxHealth` | Yes | Maximum health |
| `Healing` | No | Meta attribute: incoming heal amount |
| `Damage` | No | Meta attribute: incoming damage amount |

**Meta Attribute Mechanism:** Damage and Healing are meta attributes. They are not persisted; instead, they are converted to Health changes in `PostGameplayEffectExecute`. Damage maps to -Health, Healing maps to +Health.

### Tag Definitions

| Tag | Purpose |
|------|------|
| `TAG_Gameplay_Damage` | Generic damage tag |
| `TAG_Gameplay_DamageImmunity` | Damage immunity tag |
| `TAG_Gameplay_DamageSelfDestruct` | Self-destruct damage tag |
| `TAG_Gameplay_FellOutOfWorld` | Fall damage tag |
| `TAG_Lyra_Damage_Message` | Damage message tag |

### PreGameplayEffectExecute

Saves old values before GE execution, checks damage immunity and special damage types:

```cpp
void ULyraHealthSet::PreGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    // Save Health and MaxHealth before change
    HealthBeforeAttributeChange = Health.GetCurrentValue();
    MaxHealthBeforeAttributeChange = MaxHealth.GetCurrentValue();

    // Check Damage source
    if (Data.EvaluatedData.Attribute == GetDamageAttribute())
    {
        // Check damage immunity tag
        if (Data.Target.HasTag(TAG_Gameplay_DamageImmunity))
        {
            // Skip non-self-destruct / non-fall damage types
        }
    }
}
```

### PostGameplayEffectExecute

Converts meta attributes to actual Health changes, broadcasts change events:

```cpp
void ULyraHealthSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    if (Data.EvaluatedData.Attribute == GetDamageAttribute())
    {
        // Damage → -Health
        SetHealth(FMath::Clamp(GetHealth() - GetDamage(), 0.0f, GetMaxHealth()));
        SetDamage(0.0f);  // Zero out meta attribute
    }
    else if (Data.EvaluatedData.Attribute == GetHealingAttribute())
    {
        // Healing → +Health
        SetHealth(FMath::Clamp(GetHealth() + GetHealing(), 0.0f, GetMaxHealth()));
        SetHealing(0.0f);  // Zero out meta attribute
    }

    // Broadcast change events
    if (Data.EvaluatedData.Attribute == GetHealthAttribute())
    {
        OnHealthChanged.Broadcast(this, HealthBeforeAttributeChange, GetHealth(), ...);
    }

    // Check for death
    if (GetHealth() <= 0.0f && !bOutOfHealth)
    {
        bOutOfHealth = true;
        OnOutOfHealth.Broadcast(this);
    }
}
```

### ClampAttribute Clamping Logic

```cpp
void ULyraHealthSet::ClampAttribute(const FGameplayAttribute& Attribute, float& NewValue) const
{
    if (Attribute == GetHealthAttribute())
    {
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxHealth());
    }
    else if (Attribute == GetMaxHealthAttribute())
    {
        NewValue = FMath::Max(NewValue, 1.0f);
    }
}
```

`ClampAttribute` is called at the following points:
- `PreAttributeBaseChange()` — Before attribute base value changes
- `PreAttributeChange()` — Before attribute current value changes
- `PostAttributeChange()` — After attribute changes

### OnRep Network Replication

```cpp
void ULyraHealthSet::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(ULyraHealthSet, Health, OldHealth);
    // Trigger OnHealthChanged event on client
    OnHealthChanged.Broadcast(this, OldHealth, GetHealth(), ...);  // Instigator=nullptr
}

void ULyraHealthSet::OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(ULyraHealthSet, MaxHealth, OldMaxHealth);
    OnMaxHealthChanged.Broadcast(this, OldMaxHealth, GetMaxHealth(), ...);
}
```

## ATTRIBUTE_ACCESSORS Macro

```cpp
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)
```

This macro expands to:
- `GetXxx()` — Gets attribute value
- `SetXxx(float)` — Sets attribute value
- `InitXxx(float)` — Initializes attribute value
- `GetXxxAttribute()` — Gets attribute FProperty pointer
