# InitState Initialization State Machine

## Overview

Lyra uses a **4-state initialization state machine** to coordinate the initialization order of Pawn components (ASC, Input, Camera, Movement). Any component implementing `IGameFrameworkInitStateInterface` can register with the state machine and initialize step by step according to dependency order.

---

## Core Interface

**File**: Interface from Engine module `Components/GameFrameworkInitStateInterface.h`

```cpp
class IGameFrameworkInitStateInterface
{
public:
    // Returns this component's feature name, used for identification and dependency matching
    virtual FName GetFeatureName() const = 0;

    // Returns the state name this component expects to reach
    virtual FName GetDesiredState() const = 0;

    // Returns the "certain Feature's certain state" that this component depends on
    // Example: depends on "PawnFeature"'s "DataAvailable" state
    virtual FName GetRequiredState(const FName& FeatureName) const = 0;

    // Returns which state this component must reach before dependent components can proceed
    virtual FName GetPrerequisiteState(const FName& FeatureName) const = 0;

    // State change notification
    virtual void OnStateChanged(FName OldState, FName NewState) = 0;

    // Called before component initialization
    virtual void OnActorPreInitialize() = 0;

    // Called after component initialization state changes
    virtual void OnActorInitStateChanged(const FActorInitStateChangedParams& Params) = 0;
};
```

---

## 4-State Chain

```
Spawned
    │  Pawn created, components Registered, basic memory ready
    ▼
DataAvailable
    │  PawnExtensionComponent ready, other components can safely reference it
    ▼
DataInitialized
    │  HeroComponent ready, input binding complete, ASC can begin granting abilities
    ▼
GameplayReady
    │  All components ready, Camera and Movement fully usable
```

---

## Key Participants

| Component | File Path | Registered State | Prerequisite | Feature Name |
|-----------|-----------|-----------------|--------------|--------------|
| ULyraPawnExtensionComponent | `Character/LyraPawnExtensionComponent.h` | DataAvailable | Spawned | "PawnExtension" |
| ULyraHeroComponent | `Character/LyraHeroComponent.h` | DataInitialized | DataAvailable | "Hero" |
| UAbilitySystemComponent | Built-in GAS | GameplayReady | DataInitialized | "AbilitySystem" |
| ULyraCameraComponent | `Camera/LyraCameraComponent.h` | GameplayReady | DataInitialized | "Camera" |
| ULyraInputComponent | `Input/` | DataInitialized | DataAvailable | "Input" |

---

## Registration & Usage

### Component Registering InitState

```cpp
// Register in the component's OnRegister()
void UMyPawnComponent::OnRegister()
{
    Super::OnRegister();

    // Register with the global InitState manager
    RegisterInitStateFeature();

    // Register dependency: depends on PawnExtension's DataAvailable
    BindOnActorInitStateChanged(
        FName("PawnExtension"),
        FGameplayTag::RequestGameplayTag("InitState.DataAvailable"),
        false,
        false
    );
}
```

### State Advancement

```cpp
// Advance state when the component is ready
void UMyPawnComponent::OnActorInitStateChanged(
    const FActorInitStateChangedParams& Params)
{
    // Check if dependency is satisfied
    if (Params.FeatureName == FName("PawnExtension") &&
        Params.FeatureState == FGameplayTag::RequestGameplayTag("InitState.DataAvailable"))
    {
        // Prerequisite satisfied, initialize this component
        InitializeMyComponent();

        // Advance to target state
        CheckDefaultInitializationForFeature();
    }
}

void UMyPawnComponent::CheckDefaultInitializationForFeature()
{
    // Call framework method to advance along the state chain
    ContinueInitStateChain();
}
```

---

## State Machine Manager

The global InitState manager is implicitly provided by the engine framework, accessed through the following functions:

| Function | Description |
|----------|-------------|
| `RegisterInitStateFeature()` | Register component with the state machine |
| `UnregisterInitStateFeature()` | Unregister component |
| `CheckDefaultInitializationForFeature()` | Attempt to advance along the state chain |
| `ContinueInitStateChain()` | Explicitly advance to the next state |
| `BindOnActorInitStateChanged()` | Listen for state changes of other components |

---

## Typical Flow

```
Pawn Spawned
    → PawnExtensionComponent::OnRegister()
        → RegisterInitStateFeature()
        → State advance: Spawned → DataAvailable

    → HeroComponent::OnRegister()
        → RegisterInitStateFeature()
        → BindOnActorInitStateChanged("PawnExtension", "DataAvailable")
        → Waits for PawnExtension's DataAvailable notification

    → PawnExtensionComponent reaches DataAvailable
        → Notifies all bound components
        → HeroComponent::OnActorInitStateChanged is called
        → Initializes ASC
        → State advance: DataAvailable → DataInitialized

    → ASC initialization complete
        → Notifies bound camera and input components
        → State advance: DataInitialized → GameplayReady
```

---

## Custom Component Accessing InitState

```cpp
UCLASS()
class UMyCustomComponent : public UActorComponent,
    public IGameFrameworkInitStateInterface
{
    GENERATED_BODY()

public:
    virtual void OnRegister() override
    {
        Super::OnRegister();

        // 1. Register feature
        RegisterInitStateFeature();

        // 2. Declare dependency on PawnExtension's DataAvailable
        BindOnActorInitStateChanged(
            FName("PawnExtension"),
            FGameplayTag::RequestGameplayTag("InitState.DataAvailable"),
            false
        );
    }

    virtual FName GetFeatureName() const override
    {
        return FName("MyCustom");
    }

    virtual FName GetDesiredState() const override
    {
        return FName("GameplayReady");
    }

    // After dependency condition is met, advance own state
    virtual void OnActorInitStateChanged(
        const FActorInitStateChangedParams& Params) override
    {
        if (Params.FeatureName == FName("PawnExtension") &&
            Params.FeatureState == FGameplayTag::RequestGameplayTag("InitState.DataAvailable"))
        {
            // Initialization logic
            // ...

            ContinueInitStateChain();
        }
    }
};
```

---

## Key Design Points

1. **Dependency declaration** — Each component explicitly declares which Feature state it depends on, not relying on implicit order
2. **Asynchronous advancement** — State advances through callbacks, not dependent on Tick, supports network synchronization scenarios
3. **Extensible** — New components just need to implement the interface and register, without affecting existing components
4. **State rollback** — When a Pawn is destroyed or reused (e.g., Dormancy), state automatically rolls back to the starting point
