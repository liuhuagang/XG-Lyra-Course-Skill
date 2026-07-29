# xg-lyra-course

| Field | Value |
|-------|-------|
| Skill Name | xg-lyra-course |
| Version | UE 5.6 |
| Author | Unreal Xiaogang |
| Category | Unreal Engine Project Architecture Analysis |
| Trigger Keywords | Lyra, LyraGame, Lyra Architecture |

---

## Skill Description

### What It Can Do

- Explain the responsibility, location, and design intent of any system/class in the Lyra project
- Guide on adding new GameFeatures, weapons, equipment, abilities, UI, etc. within the Lyra framework
- Analyze cross-system call chains (e.g., the complete path from input to ability activation)
- Provide code-level references for Lyra's core design patterns (with relative paths and key line numbers)
- Diagnose common framework usage errors (e.g., InitState order, missing AbilitySet registration)
- Guide game prototype development starting from the Lyra Starter project

### What It Cannot Do

- **Does not include the actual Lyra project source code** — requires the Lyra Starter project; code paths are rooted at `Source/LyraGame/`
- **Does not cover all system details** — focuses on core architecture patterns; refer to Lyra source code for edge cases
- **Does not include Blueprint/Verse tutorials** — focuses only on the C++ layer architecture
- **Does not include in-depth network replication analysis** — only mentions key replication mechanisms

---

## Project Overview

Lyra is a **modular, data-driven multiplayer shooter framework**. Its core design principles:

- **Experience-Driven**: The entire game's loading and behavior is orchestrated by Experience data assets
- **GameFeature Plugin-Based**: Functional modules are dynamically loaded/unloaded as GameFeature Plugins
- **GAS-Centric**: All character abilities are implemented through the GameplayAbilitySystem
- **Horizontal Layering + Vertical Slicing**: Each layer (Input/GAS/Camera/UI) is independently replaceable; cross-layer communication uses GameplayTags and interfaces

### Source Directory Structure

```
Source/LyraGame/
├── AbilitySystem/              # GAS Extensions (AbilitySet, Execution, TagRelationship, Cost, etc.)
├── Camera/                     # Camera System (CameraMode, CameraAssist)
├── Character/                  # Character Class Hierarchy (LyraCharacter, LyraHeroComponent, HealthComponent)
├── Equipment/                  # Equipment System (Definition, Instance, Manager)
├── GameModes/                  # GameMode, GameState, PlayerState + Experience Framework
├── Input/                      # Input System (InputConfig, MappableConfig)
├── Inventory/                  # Inventory System (Definition, Instance, List, Manager)
├── Messages/                   # Message Protocol (VerbMessage, NotificationMessage, MessageReplication)
├── Player/                     # PlayerController, PlayerState, PlayerSpawningManager
├── Weapons/                    # Weapon System (WeaponSpawner, Spread Curves)
├── Cosmetics/                  # Cosmetics System (CharacterPart, Controller/Pawn Components)
├── Teams/                      # Teams System (TeamSubsystem, PublicInfo)
├── UI/                         # UI System (PrimaryGameLayout, HUDLayout, IndicatorSystem)
├── GameFeatures/               # GameFeature Plugins
└── Feedback/                   # Feedback System
```

Plugins/
```
Plugins/GameplayMessageRouter/   # Generic GameplayTag-based message routing subsystem
```

---

## Module Dependencies

```
Experience ──→ GameFeature Plugins (dynamically activate subsystems)
    │
    ├──→ GAS (AbilitySystemComponent + AttributeSet + GameplayAbility)
    │       │
    │       ├──→ HealthComponent (converts HealthSet attribute changes to game events)
    │       ├──→ Equipment (AbilitySet traced back via SourceObject)
    │       ├──→ Inventory (InventoryItemInstance holds AbilitySet)
    │       ├──→ Weapons (Spread/Heat curves via ILyraAbilitySourceInterface)
    │       └──→ GamePhase (GA-driven game phase transitions)
    │
    ├──→ Character
    │       ├──→ HeroComponent (coordinates Input/Camera/ASC)
    │       ├──→ PawnExtensionComponent (InitState state machine)
    │       ├──→ HealthComponent (death state machine + event dispatch)
    │       └──→ Cosmetics (CharacterPart cosmetics)
    │
    ├──→ Input ──→ EnhancedInput ──→ GAS (InputTag binding to GA)
    │
    ├──→ Camera (CameraMode stack blending)
    │
    ├──→ UI (PrimaryGameLayout 4-layer stack)
    │
    ├──→ Messages ──→ GameplayMessageRouter (Tag-driven event bus)
    │
    └──→ Teams ──→ Indicators
```

---

## Key Class Index

### Experience Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| ULyraExperienceDefinition | `Source/LyraGame/GameModes/LyraExperienceDefinition.h` | Experience data asset, defines GameFeature list and default Pawn/Controller/HUD |
| ULyraExperienceManager | `Source/LyraGame/GameModes/LyraExperienceManager.h` | Handles Experience loading state and Action execution |
| ULyraExperienceActionSet | `Source/LyraGame/GameModes/LyraExperienceActionSet.h` | Aggregates multiple UGameFeatureActions |

### Character Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| ALyraCharacter | `Source/LyraGame/Character/LyraCharacter.h` | Character base class, implements IAbilitySystemInterface, IGameplayTagAssetInterface, ILyraTeamAgentInterface |
| ULyraPawnExtensionComponent | `Source/LyraGame/Character/LyraPawnExtensionComponent.h` | Manages Pawn's InitState state machine, coordinates ASC lifecycle |
| ULyraHeroComponent | `Source/LyraGame/Character/LyraHeroComponent.h` | Player-specific component, manages input binding, camera, and ASC InitState |
| ALyraPlayerState | `Source/LyraGame/Player/LyraPlayerState.h` | Holds ASC, network owner, manages TagStack |
| ULyraHealthComponent | `Source/LyraGame/Character/LyraHealthComponent.h` | Health component, converts HealthSet attribute changes to death state machine events |
| ULyraCharacterMovementComponent | `Source/LyraGame/Character/LyraCharacterMovementComponent.h` | Movement component, Polar acceleration sync + Tag-controlled movement |

### GAS Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| ULyraAbilitySet | `Source/LyraGame/AbilitySystem/LyraAbilitySet.h` | Configurable data asset aggregating GA/GE/AttributeSet |
| ULyraGameplayAbility | `Source/LyraGame/AbilitySystem/Abilities/LyraGameplayAbility.h` | GA base class, adds ActivationGroup, AdditionalCosts, and extra trigger events |
| ULyraAbilitySystemComponent | `Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.h` | ASC extension, manages 3-phase input activation (Pressed/Held/Released) |
| ULyraAbilityCost | `Source/LyraGame/AbilitySystem/Abilities/LyraAbilityCost.h` | Abstract cost base class, CheckCost/ApplyCost interface |
| ULyraAbilityTagRelationshipMapping | `Source/LyraGame/AbilitySystem/LyraAbilityTagRelationshipMapping.h` | Cancel/Block tag relationship mapping table |
| ULyraHealthSet | `Source/LyraGame/AbilitySystem/Attributes/LyraHealthSet.h` | Health attribute set, includes ClampAttribute mechanism |
| ULyraCombatSet | `Source/LyraGame/AbilitySystem/Attributes/LyraCombatSet.h` | Combat attribute set |
| ULyraDamageExecution | `Source/LyraGame/AbilitySystem/Executions/LyraDamageExecution.h` | Custom GEEC damage calculation |
| ULyraGameplayEffectContext | `Source/LyraGame/AbilitySystem/LyraGameplayEffectContext.h` | Extended GEContext, carries CartridgeID |

### Equipment Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| ULyraEquipmentDefinition | `Source/LyraGame/Equipment/LyraEquipmentDefinition.h` | Equipment definition data asset, defines instance type and AbilitySet |
| ULyraEquipmentInstance | `Source/LyraGame/Equipment/LyraEquipmentInstance.h` | Equipment instance, runtime holds AbilitySet and SpawnedActor |
| ULyraEquipmentManagerComponent | `Source/LyraGame/Equipment/LyraEquipmentManagerComponent.h` | Equipment manager component, uses FastArray replication, manages List |
| ULyraQuickBarComponent | `Source/LyraGame/Equipment/LyraQuickBarComponent.h` | Quick bar component, manages current equipment slot switching |

### Input Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| ULyraInputConfig | `Source/LyraGame/Input/LyraInputConfig.h` | Tag→InputAction mapping data asset |
| ULyraInputComponent | `Source/LyraGame/Input/LyraInputComponent.h` | Binding helper class, auto-binds GA by Tag |

### Camera Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| ULyraCameraComponent | `Source/LyraGame/Camera/LyraCameraComponent.h` | Pawn camera component, holds CameraModeStack |
| ULyraCameraMode | `Source/LyraGame/Camera/LyraCameraMode.h` | Camera mode base class, defines FOV/distance/offset |
| ULyraCameraModeStack | `Source/LyraGame/Camera/LyraCameraMode.h` | Camera mode stack, bottom-up blending, defined in LyraCameraMode.h |
| ULyraCameraAssistInterface | `Source/LyraGame/Camera/LyraCameraAssistInterface.h` | Camera assist interface |

### UI Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| UPrimaryGameLayout | `Plugins/CommonGame/Source/Public/PrimaryGameLayout.h` | Top-level UI layout, manages 4-layer Widget stack |
| ULyraHUDLayout | `Source/LyraGame/UI/LyraHUDLayout.h` | HUD root layout, responds to PlayerState changes |
| ULyraHUD | `Source/LyraGame/UI/LyraHUD.h` | HUD base class, creates PrimaryGameLayout |

### Cosmetics Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| ULyraPawnComponent_CharacterParts | `Source/LyraGame/Cosmetics/LyraPawnComponent_CharacterParts.h` | Pawn-side cosmetics component, FastArray-driven |
| ULyraCosmeticCheats | `Source/LyraGame/Cosmetics/LyraCosmeticCheats.h` | Controller-side cosmetics authorization component |

### Teams Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| ULyraTeamSubsystem | `Source/LyraGame/Teams/LyraTeamSubsystem.h` | Team subsystem, manages PublicInfo/PrivateInfo |
| ALyraTeamPublicInfo | `Source/LyraGame/Teams/LyraTeamPublicInfo.h` | Team public info (display name, Tag), network broadcast |
| ALyraTeamPrivateInfo | `Source/LyraGame/Teams/LyraTeamPrivateInfo.h` | Team private info (server only) |
| ULyraTeamDisplayAsset | `Source/LyraGame/Teams/LyraTeamDisplayAsset.h` | Team display asset (colors, materials) |

### Indicators Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| UIndicatorDescriptor | `Source/LyraGame/UI/IndicatorSystem/IndicatorDescriptor.h` | Indicator descriptor, defines scene-to-screen mapping |
| SActorCanvas | `Source/LyraGame/UI/IndicatorSystem/SActorCanvas.h` | Slate widget, renders Indicators in screen space |

### Cheats Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| ULyraCheatManager | `Source/LyraGame/Player/LyraCheatManager.h` | CheatManager base class, supports CheatExtension |

### Player Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| ULyraPlayerSpawningManagerComponent | `Source/LyraGame/Player/LyraPlayerSpawningManagerComponent.h` | Player spawning manager, assigns spawn points by team and priority |

### Inventory Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| ULyraInventoryItemDefinition | `Source/LyraGame/Inventory/LyraInventoryItemDefinition.h` | Item definition data asset, contains Fragment array |
| ULyraInventoryItemFragment | `Source/LyraGame/Inventory/LyraInventoryItemDefinition.h` | Item fragment base class, EditInlineNew composable extension |
| ULyraInventoryItemInstance | `Source/LyraGame/Inventory/LyraInventoryItemInstance.h` | Item runtime instance, Replicated, contains StatTags |
| FLyraInventoryList | `Source/LyraGame/Inventory/LyraInventoryManagerComponent.h` | FFastArraySerializer item list |
| ULyraInventoryManagerComponent | `Source/LyraGame/Inventory/LyraInventoryManagerComponent.h` | Inventory manager component, provides Add/Remove/Consume API |
| IPickupable | `Source/LyraGame/Inventory/IPickupable.h` | Pickupable interface, returns FInventoryPickup |
| ALyraWeaponSpawner | `Source/LyraGame/Weapons/LyraWeaponSpawner.h` | Weapon spawner, supports pickup-cooldown-respawn cycle |

### Messages Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| UGameplayMessageSubsystem | `Plugins/GameplayMessageRouter/.../GameplayMessageSubsystem.h` | Message routing subsystem, GameplayTag-based publish-subscribe |
| FLyraVerbMessage | `Source/LyraGame/Messages/LyraVerbMessage.h` | Action message protocol (who did what to whom) |
| FLyraVerbMessageReplication | `Source/LyraGame/Messages/LyraVerbMessageReplication.h` | FFastArraySerializer message replication |
| UGameplayMessageProcessor | `Source/LyraGame/Messages/GameplayMessageProcessor.h` | Message processor base class, template method pattern |

### GamePhase Layer

| Class | File Path | Responsibility |
|-------|-----------|---------------|
| ULyraGamePhaseAbility | `Source/LyraGame/AbilitySystem/Phases/LyraGamePhaseAbility.h` | Phase ability, GA-driven game phase transitions |
| ULyraGamePhaseSubsystem | `Source/LyraGame/AbilitySystem/Phases/LyraGamePhaseSubsystem.h` | Phase subsystem, manages phase mutual exclusion and observer registration |

---

## Core Design Patterns

### Pattern 1: Experience Framework (3-Stage Loading)

Experience divides the game startup process into 3 stages:

```
ExperienceDefinition (data asset)
    │  Defines: GameFeature plugin list, default Pawn/Controller/HUD/GameMode
    │  Location: Content/Game/Blueprints/Experiences/
    │
    ▼
ExperienceManagerComponent (state machine)
    │  State flow: Inactive → WaitingForAction → LoadingActions → Loaded → Deactivating
    │  Path: Source/LyraGame/GameModes/LyraExperienceManagerComponent.h
    │
    ▼
UGameFeatureAction (loading action, Engine native class)
    │  Lyra aggregates multiple GameFeatureActions via ExperienceActionSet
    │  Path: indirectly referenced through ULyraExperienceActionSet
```

Lyra's ExperienceActionSet holds `TArray<TObjectPtr<UGameFeatureAction>>`, with Engine-native `UGameFeatureAction` subclasses handling plugin activation, GameMode switching, etc. You can configure GameFeature Plugin Action lists in Project Settings without defining custom Lyra-layer Action classes.

Detailed reference: [references/Experience框架与加载流程.md](references/Experience框架与加载流程.md)

---

### Pattern 2: InitState Initialization State Machine

Lyra uses a 4-state state machine to coordinate the initialization order of Pawn components (ASC/Input/Camera/Movement):

```
Spawned ──→ DataAvailable ──→ DataInitialized ──→ GameplayReady
```

**Participants**: Components implementing `IGameFrameworkInitStateInterface` (interface from Engine module `Components/GameFrameworkInitStateInterface.h`)

```
class IGameFrameworkInitStateInterface
{
    // Returns the state this Actor expects to register
    virtual FName GetFeatureName() const = 0;

    // Returns which Feature's state this Actor depends on
    virtual FName GetRequiredState() const = 0;

    // Returns the state after which other dependent Actors can proceed
    virtual FName GetPrerequisiteState() const = 0;

    // State change notification
    virtual void OnStateChanged(FName OldState, FName NewState) = 0;
};
```

**Key Implementations**:

| Component | File Path | Registered State | Prerequisite |
|-----------|-----------|-----------------|--------------|
| ULyraPawnExtensionComponent | `Character/LyraPawnExtensionComponent.h` | DataAvailable | Spawned |
| ULyraHeroComponent | `Character/LyraHeroComponent.h` | DataInitialized | DataAvailable |
| UAbilitySystemComponent | GAS Module | GameplayReady | DataInitialized |

Registration method:
```
// ULyraPawnExtensionComponent::OnRegister()
// Registers this component with the global InitState manager
RegisterInitStateFeature();
```

Detailed reference: [references/InitState初始化状态机.md](references/InitState初始化状态机.md)

---

### Pattern 3: Equipment Fragment (3-Layer Design)

The equipment system is divided into three layers with separated responsibilities:

```
ULyraEquipmentDefinition (Definition Layer)
    │  As PrimaryDataAsset, defines Instance type and AbilitySet
    │  Path: Source/LyraGame/Equipment/LyraEquipmentDefinition.h
    │
    ▼
ULyraEquipmentInstance (Instance Layer)
    │  Runtime instance, holds GrantedAbilitySet and SpawnedActor
    │  Path: Source/LyraGame/Equipment/LyraEquipmentInstance.h
    │
    ▼
ULyraEquipmentManagerComponent (Manager Layer)
    │  FastArray sync list (FLyraEquipmentList)
    │  Path: Source/LyraGame/Equipment/LyraEquipmentManagerComponent.h
```

AbilitySet grant chain after equipping:
```
EquipmentManagerComponent::AddEquipment()
    → EquipmentInstance.SpawnEquipmentActor()
    → EquipmentDefinition.GrantedAbilities (reads AbilitySet)
    → ASC::GiveAbility() / ASC::InitStats()
    → EquipmentInstance records Granted Handles (for removal/cleanup)
```

**SourceObject Tracing**: GA can trace back to EquipmentInstance via `GetSourceObject()`, and further to EquipmentDefinition, allowing GearContext to carry equipment information (e.g., CartridgeID).

```
// Getting the source equipment in GA
ULyraEquipmentInstance* EquipInst = Cast<ULyraEquipmentInstance>(
    GetAbilitySourceObject()
);
```

Detailed reference: [references/装备与武器系统.md](references/装备与武器系统.md)

---

### Pattern 4: Input Config (Tag→Action Mapping)

The input system uses a **Tag-driven** approach to decouple input from GameplayAbility:

```
EnhancedInput InputAction
    │  ULyraInputConfig maintains Tag→InputAction mapping
    │  Path: Source/LyraGame/Input/LyraInputConfig.h
    ▼
InputTag (e.g., InputTag.Move, InputTag.Jump)
    │  ULyraInputComponent auto-binds by Tag
    ▼
GameplayAbility (activated via InputTag)
```

```
// ULyraInputConfig core structure
UCLASS()
class ULyraInputConfig : public UDataAsset
{
    GENERATED_BODY()

public:
    // Tag→InputAction mapping
    UPROPERTY(EditDefaultsOnly, Category = "Input")
    TArray<FLyraInputAction> InputActions;

    // Find InputAction by Tag
    const UInputAction* FindInputActionForTag(const FGameplayTag& Tag) const;
};

USTRUCT()
struct FLyraInputAction
{
    UPROPERTY(EditDefaultsOnly)
    FGameplayTag InputTag;

    UPROPERTY(EditDefaultsOnly)
    TObjectPtr<UInputAction> InputAction;
};
```

Binding to GA:
```
// ULyraHeroComponent::InitializePlayerInput()
// Path: Source/LyraGame/Character/LyraHeroComponent.h

ULyraInputComponent* LyraInputComp = ...;
ULyraInputConfig* InputConfig = ...;

// Bind GA by Tag
LyraInputComp->BindAbilityActions(
    InputConfig,                 // Tag→Action mapping
    InputHandles,                // Output handles
    this,                        // UObject
    &ThisClass::OnInputStarted,
    &ThisClass::OnInputTriggered,
    &ThisClass::OnInputCompleted
);
```

Detailed reference: [references/输入系统与InputConfig.md](references/输入系统与InputConfig.md)

---

### Pattern 5: Camera Stack (3-Layer Camera Architecture)

The camera system is divided into three layers:

```
ULyraCameraComponent (Component Layer)
    │  Attached to Pawn, holds CameraModeStack
    │  Path: Source/LyraGame/Camera/LyraCameraComponent.h
    │
    ▼
ULyraCameraModeStack (Stack Layer)
    │  Manages active CameraMode list, bottom-up blending
    │  Path: Source/LyraGame/Camera/LyraCameraMode.h (defined in same file)
    │
    ▼
ULyraCameraMode (Mode Layer)
    │  Defines specific camera behavior (FOV/distance/offset/spring arm)
    │  Path: Source/LyraGame/Camera/LyraCameraMode.h
```

Blend calculation process:
```
CameraModeStack::GetBlendInfo()
    → Iterates through active CameraMode list
    → Blends from bottom mode0 to top modeN
    → Final BlendInfo = mode0 * (1-blend) + mode1 * blend
```

```cpp
// Adding a custom CameraMode
void AMyPawn::SetupCustomCamera()
{
    ULyraCameraComponent* CameraComp = FindComponentByClass<ULyraCameraComponent>();
    if (CameraComp)
    {
        // Default mode: third person
        CameraComp->AddCameraModeClass(ULyraCameraMode_ThirdPerson::StaticClass());

        // Overlay mode: aim down sights zoom
        CameraComp->AddCameraModeClass(ULyraCameraMode_AimDownSights::StaticClass());
    }
}
```

---

### Pattern 6: UI Layer Stack (4-Layer UI Stack)

The UI system uses `UPrimaryGameLayout` to manage a 4-layer Widget stack, each layer independent:

```
Game (Game Layer)       — HUD, crosshair, health bar, skill cooldowns
GameMenu (Menu Layer)    — In-game menu (ESC menu)
Menu (Feature Layer)     — Inventory, Settings, Shop
Modal (Modal Layer)      — Prompt dialogs, confirmation dialogs (BlockInput)
```

```
// Path: Plugins/CommonGame/Source/Public/PrimaryGameLayout.h
UCLASS()
class UPrimaryGameLayout : public UCommonActivatableWidgetStack
{
    GENERATED_BODY()

public:
    // 4-layer stack access
    UCommonActivatableWidgetStack* GetGameStack() const;
    UCommonActivatableWidgetStack* GetGameMenuStack() const;
    UCommonActivatableWidgetStack* GetMenuStack() const;
    UCommonActivatableWidgetStack* GetModalStack() const;
};
```

Pushing a Widget to a specific layer:
```
// Push HUD to Game layer
UPrimaryGameLayout* RootLayout = UPrimaryGameLayout::GetPrimaryGameLayout(Controller);
RootLayout->GetGameStack()->AddWidget(CreateWidget<UMyHUDWidget>(Controller));
```

Default HUD structure in Lyra:
```
ALyraHUD (Path: Source/LyraGame/UI/LyraHUD.h)
    └── ULyraHUDLayout (Root Widget)
            ├── TopLeft Tier (Team info, player list)
            ├── LowerLeft Tier (Health, ammo)
            ├── LowerMiddle Tier (Quick bar, equipment switching)
            └── LowerRight Tier (Kill feed, notifications)
```

Detailed reference: [references/UI层栈架构.md](references/UI层栈架构.md)

---

### Pattern 7: HealthComponent Death State Machine

The Health component acts as a bridge between the GAS attribute system and the game presentation layer, with a core 3-state death state machine:

```
NotDead ──→ DeathStarted ──→ DeathFinished
```

**File**: `Source/LyraGame/Character/LyraHealthComponent.h`

```
ULyraHealthComponent
  InitializeWithAbilitySystem(ASC)
    → HealthSet's AttributeChangeDelegate binding
    → OnHealthChanged / OnMaxHealthChanged
    → StartDeath / FinishDeath state control

3-state death state machine:
  NotDead (normal state)
    → Receives GE damage, Health reaches zero
    → StartDeath → bDead = true, state = DeathStarted
    → OnDeathStarted.Broadcast()

  DeathStarted (death trigger started)
    → Triggers ragdoll, death animation, etc.
    → Delayed call to FinishDeath → state = DeathFinished
    → OnDeathFinished.Broadcast()

  DeathFinished (death completed)
    → Character fully dead, responds to Ragdoll end or respawn
```

Complete call chain:

```
GE Damage → ULyraDamageExecution::Execute
  → HealthSet::PostAttributeChange (Health zero detection)
  → ULyraHealthComponent::OnHealthChanged
  → StartDeath() → bDead = true
  → OnDeathStarted.Broadcast()
  → ULyraCharacter::OnDeathStarted (ragdoll)
  → Delay → FinishDeath()
  → OnDeathFinished.Broadcast()
```

4 core Delegates:

| Delegate | Trigger | Usage |
|----------|---------|-------|
| OnHealthChanged | Health attribute changes | UI health bar update |
| OnMaxHealthChanged | MaxHealth attribute changes | UI health bar max adjustment |
| OnDeathStarted | Entering DeathStarted state | Play death animation/ragdoll |
| OnDeathFinished | Entering DeathFinished state | Respawn/scoring/cleanup |

Detailed reference: [references/GAS能力系统架构.md](references/GAS能力系统架构.md)

---

### Pattern 8: GameplayMessageRouter Publish-Subscribe

The message routing system provides a global event bus based on GameplayTag, enabling loosely coupled cross-system communication.

**Plugin Core** (GameplayMessageRouter Plugin):

```
// Broadcast message: iterate all matching Listeners by Tag
BroadcastMessage(Tag, Message)
  → Exact Tag match + parent tag fallback

// Register listener
RegisterListener(Params)
  → Params include Tag, callback function, match pattern
```

**Application Layer Protocol**: FLyraVerbMessage

Describes "who (Instigator) did what (Verb) to whom (Target)":

```
struct FLyraVerbMessage {
    FGameplayTag Verb;          // Action tag
    TObjectPtr<UObject> Instigator;
    TObjectPtr<UObject> Target;
    FGameplayTagContainer InstigatorTags;
    FGameplayTagContainer TargetTags;
    FGameplayTagContainer ContextTags;
    double Magnitude;
};
```

**Message Replication Mechanism**: FLyraVerbMessageReplication

```
Server
  AddMessage(VerbMessage)
  → FastArray (FLyraVerbMessageReplication) auto NetDeltaSerialize
  → Client PostReplicatedAdd/Change
    → Local re-BroadcastMessage
    → Local listeners receive message
```

**Message Processor Base Class**: UGameplayMessageProcessor

Template method pattern, subclasses only need to override StartListening():

```
BeginPlay() → StartListening()
EndPlay()   → StopListening() (auto-cleanup ListenerHandle)
```

Detailed reference: [references/库存与消息系统.md](references/库存与消息系统.md)

---

## Cross-System Call Chains

### Chain 1: Player Input → Ability Activation

```
Player presses key
    → EnhancedInput triggers UInputAction
    → ULyraHeroComponent::OnInputStarted (InputTag matching)
        [Path: Source/LyraGame/Character/LyraHeroComponent.h]
    → ASC::AbilityLocalInputPressed (Tag mapped to GA)
    → ULyraGameplayAbility::ActivateAbility
        [Path: Source/LyraGame/AbilitySystem/Abilities/LyraGameplayAbility.h]
    → GA internally calls CommitAbility / ApplyGameplayEffect
    → ULyraDamageExecution::Execute_Implementation (damage calculation)
        [Path: Source/LyraGame/AbilitySystem/Executions/LyraDamageExecution.h]
```

### Chain 2: Experience Loading → Character Initialization

```
GameMode::InitGame
    → Loads ExperienceDefinition
    → ExperienceManagerComponent::StartExperience
        [Path: Source/LyraGame/GameModes/LyraExperienceManagerComponent.h]
    → Activates GameFeature plugins
    → Spawn Default Pawn
    → Pawn::PostInitializeComponents
    → PawnExtensionComponent::OnRegister (registers InitState)
        [Path: Source/LyraGame/Character/LyraPawnExtensionComponent.h]
    → InitState: Spawned → DataAvailable → DataInitialized → GameplayReady
    → Components ready in order
```

### Chain 3: Equipment Switch → AbilitySet Change

```
Player presses number key
    → ULyraQuickBarComponent::CycleNextSlot / SetActiveSlot
        [Path: Source/LyraGame/Equipment/LyraQuickBarComponent.h]
    → EquipmentManagerComponent::RemoveEquipment (unequip old equipment)
        [Path: Source/LyraGame/Equipment/LyraEquipmentManagerComponent.h]
    → ASC::RemoveAbility (reclaim old GA handles)
    → EquipmentManagerComponent::AddEquipment (equip new equipment)
    → SpawnEquipmentActor
    → ASC::GiveAbility / InitStats (grant new GA and attributes)
    → New equipment's ULyraEquipmentInstance records Granted Handles
```

### Chain 4: Cosmetics Sync (Network)

```
Server initiates cosmetics change
    → ULyraCosmeticCheats::AddCharacterPart
        [Path: Source/LyraGame/Cosmetics/LyraCosmeticCheats.h]
    → ULyraPawnComponent_CharacterParts::AddCharacterPart
        [Path: Source/LyraGame/Cosmetics/LyraPawnComponent_CharacterParts.h]
    → FastArray (FLyraAppliedCharacterPartList) replicates to clients
    → OnRep_CharacterPartList triggers
    → Spawn/Create corresponding CharacterPart Actor
```

### Chain 5: Damage → Death Handling

```
GA activates → ApplyGameplayEffect
    → ULyraDamageExecution::Execute_Implementation
        [Path: Source/LyraGame/AbilitySystem/Executions/LyraDamageExecution.h]
    → HealthSet Health attribute reaches zero
        [Path: Source/LyraGame/AbilitySystem/Attributes/LyraHealthSet.h]
    → ULyraHealthComponent::OnHealthChanged (attribute change callback)
        [Path: Source/LyraGame/Character/LyraHealthComponent.h]
    → StartDeath() → bDead = true, state = DeathStarted
    → OnDeathStarted.Broadcast()
    → ALyraCharacter::OnDeathStarted (play ragdoll/death animation)
    → Delay → FinishDeath() → state = DeathFinished
    → OnDeathFinished.Broadcast()
    → Optional: Send FLyraVerbMessage (kill notification)
      → GameplayMessageSubsystem::BroadcastMessage
```

### Chain 6: Pickup → Inventory → Equipment

```
Player enters ALyraWeaponSpawner collision
    [Path: Source/LyraGame/Weapons/LyraWeaponSpawner.h]
    → AttemptPickUpWeapon(Pawn)
    → GiveWeapon(WeaponItemClass, InventoryManager)
    → ULyraInventoryManagerComponent::AddItemDefinition
        [Path: Source/LyraGame/Inventory/LyraInventoryManagerComponent.h]
    → FLyraInventoryList::AddEntry (FastArray sync to client)
    → ULyraQuickBarComponent detects new item
        [Path: Source/LyraGame/Equipment/LyraQuickBarComponent.h]
    → Switches to new slot
    → ULyraEquipmentManagerComponent::AddEquipment
        [Path: Source/LyraGame/Equipment/LyraEquipmentManagerComponent.h]
    → ULyraEquipmentInstance::SpawnEquipmentActor
    → ULyraAbilitySet::GiveToAbilitySystem (grants abilities and attributes)
```

---

## Common Development Workflows

### Workflow 1: Adding a New Weapon

1. Create a weapon class inheriting `ULyraEquipmentInstance`, path: `Equipment/`
2. Create an AbilitySet data asset containing weapon GA, GE, AttributeSet
3. Create a `ULyraEquipmentDefinition` data asset pointing to steps 1 and 2
4. Configure the Definition into an inventory item or test directly via QuickBarComponent
5. (Optional) Create a CameraMode subclass for the weapon to implement aim zoom

### Workflow 2: Adding a New Ability (GA)

1. Inherit `ULyraGameplayAbility`, path: `AbilitySystem/Abilities/`
2. Set `ActivationGroup` (Independent / Exclusive_Replaceable / Exclusive_Blocking)
3. Define `ActivationOwnedTags` and `ActivationRequiredTags`
4. If the ability involves damage, use `FLyraGameplayEffectContext` to carry CartridgeID
5. Add to a `ULyraAbilitySet` data asset

### Workflow 3: Adding a New GameMode

1. Create a new `ULyraExperienceDefinition` data asset
2. Define the GameFeature plugin list loaded by this Experience
3. Set default Pawn, Controller, HUD, PlayerState, GameState
4. Configure GameMode switching in the ExperienceDefinition's GameFeature Actions
5. Point Project Settings → Maps & Modes → Default GameMode to the custom GameMode

### Workflow 4: Adding a New UI Screen

1. Create a Widget inheriting `UCommonActivatableWidget` (supports Push/Pop lifecycle)
2. Choose the appropriate layer in `UPrimaryGameLayout`'s 4-layer stack
3. Call `RootLayout->GetXxxStack()->AddWidget(Widget)` to display
4. Implement `OnActivated`/`OnDeactivated` lifecycle methods

### Workflow 5: Adding a New Team Display Asset

1. Create a `ULyraTeamDisplayAsset` data asset, set colors and materials
2. Set DisplayAsset on `ALyraTeamPublicInfo`
3. Each system's display logic reads DisplayAsset from TeamSubsystem to update visuals

### Workflow 6: Adding an Inventory Item

1. Create a `ULyraInventoryItemDefinition` data asset, set DisplayName
2. Add `ULyraInventoryItemFragment` subclasses to extend item behavior (e.g., weapon fragment, consumable fragment)
3. If the item needs to be pickupable, configure `ALyraWeaponSpawner` and point its `ULyraWeaponPickupDefinition` to the item definition
4. Admin calls `InventoryManagerComponent->AddItemDefinition(ItemDef, StackCount)`
5. The item is added to `FLyraInventoryList` (FastArray auto-sync)

### Workflow 7: Adding Damage Feedback and Death Presentation

1. Ensure the character has `ULyraHealthComponent` and `ULyraHealthSet` configured (typically provided by the PawnData's AbilitySet by default)
2. Create a custom GE Blueprint, set damage attributes (Damage.Health), damage type Tag
3. Bind `OnHealthChanged` event in the character Blueprint to display damage numbers, HitMarker
4. Bind `OnDeathStarted` event in the character Blueprint to play death montage/ragdoll
5. Bind `OnDeathFinished` event in the character Blueprint to handle respawn/scoring
6. (Optional) Send kill notification via `FLyraVerbMessage`, UI layer listens to display kill feed

---

## Extended Reading

| Document | Content |
|----------|---------|
| [Experience Framework and Loading Flow](references/Experience框架与加载流程.md) | Experience 3-stage loading, GameFeature activation flow, ManagerComponent state machine |
| [InitState Initialization State Machine](references/InitState初始化状态机.md) | 4-state state machine, IGameFrameworkInitStateInterface, component registration |
| [Input System and InputConfig](references/输入系统与InputConfig.md) | InputConfig data structure, BindAbilityActivation, input modifiers |
| [GAS Ability System Architecture](references/GAS能力系统架构.md) | AbilitySet, ActivationGroup, ASC input activation, HealthComponent death state machine, AbilityCost |
| [Equipment and Weapon System](references/装备与武器系统.md) | 3-layer equipment design, QuickBarComponent, Spread/Heat mechanism |
| [UI Layer Stack Architecture](references/UI层栈架构.md) | 4-layer UI stack, PrimaryGameLayout, HUDLayout Tier structure |
| [Animation System Overview](references/动画系统概览.md) | Lyra animation architecture |
| [Character Initialization and Movement System](references/角色初始化与移动系统.md) | PlayerSpawningManager, movement acceleration Polar quantization, Tag-controlled movement |
| [Inventory and Message System](references/库存与消息系统.md) | 4-layer inventory architecture, Pickup system, GameplayMessageRouter, GamePhase |

---

## Usage Examples

### Example: Asking "How do I add a rocket launcher weapon?"

> To add a rocket launcher weapon to the Lyra project:
>
> 1. **Create EquipmentDefinition**: Create a `ULyraEquipmentDefinition` data asset in the Content directory, set Instance type and AbilitySet
> 2. **Create EquipmentInstance**: Inherit `ULyraEquipmentInstance` to add rocket launcher-specific logic (e.g., SpawnedActor type)
> 3. **Create AbilitySet**: Include rocket launcher fire GA, explosion GE, rocket ammo AttributeSet
> 4. **Configure QuickBarComponent**: Add the new weapon's Definition to QuickBarComponent's DefaultSlots
> 5. **Add Spread Curve**: For spread mechanics, implement the `ILyraAbilitySourceInterface` interface in the weapon GA
>
> Reference code pattern: Equipment Fragment 3-layer design (see Core Design Pattern 3)

### Example: Asking "Explain the cosmetics synchronization mechanism in Lyra"

> Lyra's cosmetics system uses a **dual-component architecture**:
>
> - **ULyraCosmeticCheats** (attached to Controller): Authoritative end, only the server can modify the cosmetics list
> - **ULyraPawnComponent_CharacterParts** (attached to Pawn): Cosmetics data replicated via FastArray
>
> When the server calls `AddCharacterPart`, it modifies `FLyraAppliedCharacterPartList` (FastArray), which is automatically replicated to all clients. Upon receiving `OnRep`, clients spawn the corresponding `UCharacterPart` Actor (e.g., weapon attachments, helmets as static meshes).
>
> Key file: `Source/LyraGame/Cosmetics/`
