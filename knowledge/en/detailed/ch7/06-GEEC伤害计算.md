# GEEC Damage Calculation

> Corresponding lecture: [086_讲解GEEC](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/docs/UE5_Lyra学习指南_086_讲解GEEC.md)
> Core code: [LyraDamageExecution](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/Executions/LyraDamageExecution.h), [LyraHealExecution](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/Executions/LyraHealExecution.h)

## ULyraDamageExecution

Inherits from `UGameplayEffectExecutionCalculation`, implementing custom damage calculation.

### FDamageStatics

Private static struct, caches attribute capture definitions:

```cpp
struct FDamageStatics
{
    // Declare attribute capture
    DECLARE_ATTRIBUTE_CAPTUREDEF(BaseDamage);  // Captured from Source CombatSet

    FDamageStatics()
    {
        DEFINE_ATTRIBUTE_CAPTUREDEF(ULyraCombatSet, BaseDamage, Source, true);
    }
};

static const FDamageStatics& GetDamageStatics()
{
    static FDamageStatics Statics;
    return Statics;
}
```

### Execute_Implementation Flow

```
ULyraDamageExecution::Execute_Implementation(ExecutionParams, OutExecutionOutput)
  ├── 1. Capture BaseDamage
  │     └── ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
  │           GetDamageStatics().BaseDamageDef, EvaluationParams, BaseDamage)
  │
  ├── 2. Get EffectContext
  │     └── FLyraGameplayEffectContext::ExtractEffectContext(Params.GetEffectContext())
  │           └── Extract HitResult, AbilitySource
  │
  ├── 3. Calculate team damage multiplier
  │     ├── Get target Actor from HitResult
  │     ├── Query team subsystem (ULyraTeamSubsystem)
  │     │     └── Compare Source and Target team IDs
  │     ├── If same team: DamageInteractionAllowedMultiplier = 0.0 (friendly fire disabled)
  │     └── If enemy team: DamageInteractionAllowedMultiplier = 1.0 (normal damage)
  │
  ├── 4. Calculate attenuation
  │     ├── Get ILyraAbilitySourceInterface from AbilitySource
  │     ├── Calculate distance attenuation
  │     │     └── AbilitySource->GetDistanceAttenuation(Distance, SourceTags, TargetTags)
  │     └── Calculate physical material attenuation
  │           └── AbilitySource->GetPhysicalMaterialAttenuation(PhysMat, SourceTags, TargetTags)
  │
  ├── 5. Final damage formula
  │     └── FinalDamage = BaseDamage × DistanceAttenuation × PhysicalMaterialAttenuation × DamageInteractionAllowedMultiplier
  │
  └── 6. Output damage
        └── OutExecutionOutput.AddOutputModifier(
              FGameplayModifierEvaluatedData(GetDamageAttribute(), EGameplayModOp::Additive, FinalDamage))
```

### Final Damage Formula

```
FinalDamage = BaseDamage × DistanceAttenuation × PhysicalMaterialAttenuation × DamageInteractionAllowedMultiplier
```

| Factor | Source | Description |
|------|------|------|
| BaseDamage | CombatSet | Base damage value of weapon/ability |
| DistanceAttenuation | AbilitySource interface | Distance-based attenuation curve |
| PhysicalMaterialAttenuation | AbilitySource interface | Physical material-based attenuation coefficient |
| DamageInteractionAllowedMultiplier | Team subsystem | 0.0 (friendly) / 1.0 (enemy) |

## ULyraHealExecution

Heal execution calculation, structurally symmetric to ULyraDamageExecution.

### FHealStatics

```cpp
struct FHealStatics
{
    DECLARE_ATTRIBUTE_CAPTUREDEF(BaseHeal);  // Captured from Source CombatSet

    FHealStatics()
    {
        DEFINE_ATTRIBUTE_CAPTUREDEF(ULyraCombatSet, BaseHeal, Source, true);
    }
};
```

### Execution Flow

```
ULyraHealExecution::Execute_Implementation(ExecutionParams, OutExecutionOutput)
  ├── Capture BaseHeal
  └── Output Healing attribute
        └── No attenuation or team check involved
```

Note: ULyraHealExecution is wrapped in `#if WITH_SERVER_CODE`, compiled only on server.

## FLyraGameplayEffectContext's Role in GEEC

FLyraGameplayEffectContext serves as the data transfer object in GEEC:

```
GE Spec → FLyraGameplayEffectContext
  ├── HitResult (hit info, including PhysMat)
  ├── AbilitySourceObject (weapon/ability source)
  └── CartridgeID (bullet batch ID)
        └── Used after Extract in GEEC:
              ├── HitResult → Get target Actor, physical material, distance
              ├── AbilitySource → Get attenuation coefficients
              └── → Feed into final damage calculation
```
