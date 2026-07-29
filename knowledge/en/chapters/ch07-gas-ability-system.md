# Chapter 7: GAS Ability System

> Corresponding lectures: 080~090 | Source directories: AbilitySystem/, Interaction/, Inventory/

## Knowledge Map

| Knowledge File | Corresponding Lecture | Description |
|----------------|-----------------------|-------------|
| [ch7/01-GAS架构概述](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch7/01-GAS架构概述.md) | 080 | AbilitySet, AbilitySystemGlobals, EffectContext, GlobalAbilitySystem, AbilityTagRelationshipMapping, AbilityCost overview |
| [ch7/02-ASC与输入激活](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch7/02-ASC与输入激活.md) | 081 | ASC input pipeline, ProcessAbilityInput, activation group, tag relationships, ability failure, dynamic GE, camera mode |
| [ch7/03-GA_Jump与GA_Dash](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch7/03-GA_Jump与GA_Dash.md) | 082, 083 | Jump ability (CharacterMovement driven), Dash ability (RootMotion driven), AbilityTask details |
| [ch7/04-GA_Melee](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch7/04-GA_Melee.md) | 084 | Melee ability: Capsule Trace, friendly check, damage GEEC, Lunge |
| [ch7/05-属性集与HealthSet](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch7/05-属性集与HealthSet.md) | 085 | Attribute set hierarchy, HealthSet, meta-attribute mechanism, ClampAttribute, ATTRIBUTE_ACCESSORS |
| [ch7/06-GEEC伤害计算](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch7/06-GEEC伤害计算.md) | 086 | DamageExecution, HealExecution, capture/attenuation/team damage, damage formula |
| [ch7/07-生命值组件](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch7/07-生命值组件.md) | 087 | HealthComponent, death state machine, attribute event bridging, network replication |
| [ch7/08-库存系统](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch7/08-库存系统.md) | 088 | InventoryManager, ItemDefinition/Fragment/Instance, FFastArraySerializer, IPickupable |
| [ch7/09-交互系统](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch7/09-交互系统.md) | 089, 090 | IInteractableTarget, FInteractionOption, ALyraWorldCollectable, pickup flow |

## Core Architecture Diagram

```
Input Tag
  │
  v
ULyraHeroComponent ──> ULyraAbilitySystemComponent ──> ProcessAbilityInput()
  │                             │
  │                    TagRelationshipMapping
  │                    ActivationGroup
  │                    GlobalAbilitySystem
  │
  v
ULyraGameplayAbility
  ├── GA_Jump (CharacterMovement)
  ├── GA_Dash (RootMotion)
  ├── GA_Melee (Capsule Trace)
  └── GA_Interact (Interaction)
        │
        v
  ULyraAbilitySet (PawnData → PlayerState → GiveToAbilitySystem)
        │
  ULyraDamageExecution / ULyraHealExecution (GEEC)
        │
  FLyraGameplayEffectContext (DamageAttenuation)
        │
  ULyraHealthSet ──> ULyraHealthComponent ──> DeathState
        │
  ULyraInventoryManagerComponent
        ├── FLyraInventoryList (FFastArraySerializer)
        ├── ULyraInventoryItemDefinition (Fragments)
        └── ULyraInventoryItemInstance (StatTags)
        │
  ALyraWorldCollectable (IInteractableTarget + IPickupable)
```

## Key Mechanisms Overview

| Mechanism | Description | Key File |
|-----------|-------------|----------|
| Input Activation | EnhancedInput → HeroComponent → ASC::AbilityInputTagPressed → ProcessAbilityInput | [02](../detailed/ch7/02-ASC与输入激活.md) |
| Activation Group | Independent / Exclusive_Replaceable / Exclusive_Blocking | [02](../detailed/ch7/02-ASC与输入激活.md) |
| Tag Relationships | AbilityTag → Block/Cancel/Required/Blocked Tags | [01](../detailed/ch7/01-GAS架构概述.md) |
| Ability Granting | PawnData → AbilitySet → PlayerState::SetPawnData | [01](../detailed/ch7/01-GAS架构概述.md) |
| Attribute Set | HealthSet meta-attribute pattern (Damage→-Health, Healing→+Health) | [05](../detailed/ch7/05-属性集与HealthSet.md) |
| Damage Calculation | GEEC: BaseDamage × DistanceAtten × PhysMatAtten × TeamMultiplier | [06](../detailed/ch7/06-GEEC伤害计算.md) |
| Death Flow | NotDead → DeathStarted → DeathFinished | [07](../detailed/ch7/07-生命值组件.md) |
| Item System | Definition/Fragment (design-time) → Instance (runtime) → List (replication) | [08](../detailed/ch7/08-库存系统.md) |
| Interaction Pickup | IInteractableTarget::GatherInteractionOptions → IPickupable::GetPickupInventory | [09](../detailed/ch7/09-交互系统.md) |
