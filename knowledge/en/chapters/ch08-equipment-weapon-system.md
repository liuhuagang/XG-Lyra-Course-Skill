# Chapter 8: Equipment & Weapon System

This chapter covers Lyra's equipment and weapon system, from the equipment framework to ranged weapon firing flow, damage feedback, and aim assist.

## Chapter Content Index

### Equipment & Pickup

- **[Equipment System Architecture](../detailed/ch8/装备系统架构.md)**
  - EquipmentDefinition/Instance/ManagerComponent three-tier design
  - QuickBar equipment flow
  - FLyraEquipmentList network replication
  - PickupDefinition pickup definition

- **[Weapon Spawner & Pickup System](../detailed/ch8/武器生成器与拾取系统.md)**
  - ALyraWeaponSpawner weapon spawner
  - Pickup/cooldown/respawn flow
  - Pickup definition data asset

### Weapon System

- **[Ranged Weapon Definition & Spread System](../detailed/ch8/远程武器定义与扩散系统.md)**
  - ULyraWeaponInstance / ULyraRangedWeaponInstance
  - Heat/Spread curve system
  - Multi-state precision multipliers (Standing/Crouching/Jump/Aim)
  - First Shot Accuracy
  - Distance damage attenuation and material damage multipliers

- **[Weapon Fire Ability & Prediction System](../detailed/ch8/武器开火技能与预测系统.md)**
  - ULyraGameplayAbility_RangedWeapon
  - Dual-stage bullet trace (Ray → Sweep)
  - Spread direction calculation (VRandConeNormalDistribution)
  - Prediction system (FScopedPredictionWindow / TargetData)

### Reticle & Feedback

- **[Weapon Reticle & Hit Marker System](../detailed/ch8/武器准心与命中标记系统.md)**
  - ULyraWeaponStateComponent hit tracking
  - Reticle spread rendering (ULyraReticleWidgetBase)
  - Circumference marker widget (UCircumferenceMarkerWidget)
  - Hit marker confirmation (SHitMarkerConfirmationWidget)

- **[Damage Feedback & Context Effects System](../detailed/ch8/伤害反馈与上下文效果系统.md)**
  - Context Effects framework
  - Physical material-driven sound system
  - Camera shake and haptic feedback

- **[Damage Number Popup System](../detailed/ch8/伤害数字弹出系统.md)**
  - NiagaraText / MeshText two rendering methods
  - NumberPop trigger flow

### Extended Abilities

- **[Grenade Ability](../detailed/ch8/手榴弹技能.md)**
  - Parabolic throw ability

- **[Aim Assist System](../detailed/ch8/辅助射击系统.md)**
  - Drag/Tension
  - Slowdown

## Core Architecture Diagram

```
Equipment System                  Weapon System
┌─────────────────┐           ┌──────────────────────────┐
│ EquipmentDef    │           │ WeaponInstance           │
│ (Definition/    │           │ (Animation layer/        │
│  AbilitySet)    │           │  device properties)      │
│       ↓         │           │       ↓                  │
│ EquipmentInst   │           │ RangedWeaponInstance     │
│ (Runtime        │           │ (Spread/Heat curves)     │
│  instance)      │           │       ↓                  │
│       ↓         │           │ GA_RangedWeapon          │
│ EquipMgrComp    │           │ (Fire/Trace/Prediction)  │
│ (FastArray)     │           └──────────────────────────┘
└─────────────────┘

Reticle & Feedback               Pickup System
┌──────────────────┐           ┌──────────────────────────┐
│ WeaponStateComp  │           │ WeaponSpawner            │
│ (Hit tracking)   │           │ (Scene spawner)          │
│ ReticleWidget    │           │ Cooldown/Respawn flow    │
│ HitMarkerWidget  │           │ PickupDef DataAsset      │
└──────────────────┘           └──────────────────────────┘
```
