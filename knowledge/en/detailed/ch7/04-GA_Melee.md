# GA_Melee (Melee Ability)

> Corresponding lecture: [084_讲解GA_Melee](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/docs/UE5_Lyra学习指南_084_讲解GA_Melee.txt)

GA_Melee is a Blueprint-based melee attack ability that uses Capsule Trace as its hit detection method.

## Ability Configuration

| Property | Value |
|------|-----|
| InputTag | `InputTag.Ability.Melee` |
| NetExecutionPolicy | LocalPredicted |
| InstancingPolicy | InstancedPerActor |

## Execution Flow

```
InputTag.Ability.Melee Pressed
  └── TryActivateAbility()
        └── ActivateAbility()
              ├── Play melee animation Montage
              │     ├── Swing attack animation
              │     └── Lunge animation — forward displacement
              │
              ├── Hit detection (Capsule Trace)
              │     ├── Front fan-shaped area sweep
              │     ├── Actor detected → Check friendly fire
              │     └── Obstruction check
              │
              ├── On Hit
              │     ├── Apply damage GE (via GEEC)
              │     ├── Play hit sound
              │     └── Play impact effects
              │
              ├── On Miss
              │     └── Play swing sound
              │
              └── EndAbility()
```

## Damage Calculation

Melee damage uses the same GEEC (ULyraDamageExecution) as ranged weapons, so the damage formula is consistent:

```
Final Damage = BaseDamage × DistanceAttenuation × PhysicalMaterialAttenuation × DamageInteractionAllowedMultiplier
```

Melee distance attenuation (DistanceAttenuation) is typically 1.0 (melee attenuation coefficient is 1), and physical material attenuation is determined by the hit location's physical material.

## Lunge Mechanism

When performing a melee attack, the character lunges forward a distance to close in on the target. The Lunge movement is driven by Root Motion in the Montage, unlike GA_Dash which uses `ApplyRootMotionConstantForce`.

## Friendly Fire Check

Filtering steps after hit detection:
1. Actor detected
2. Get Team info on target (via team subsystem)
3. If friendly (same team), skip damage application
4. If enemy or neutral, execute damage GE

## Sound Management

| Event | Sound |
|------|------|
| Swing attack | Swing Sound |
| Hit target | Hit Sound |
| Lunge | Lunge Sound |
