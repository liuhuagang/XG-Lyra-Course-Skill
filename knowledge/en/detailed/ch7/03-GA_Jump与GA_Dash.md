# GA_Jump & GA_Dash

> Corresponding lectures: [082_讲解GA_Jump](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/docs/UE5_Lyra学习指南_082_讲解GA_Jump.md), [083_讲解GA_Dash](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/docs/UE5_Lyra学习指南_083_讲解GA_Dash.md)
> Core code: [LyraGameplayAbility_Jump](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/Abilities/LyraGameplayAbility_Jump.h)

## ULyraGameplayAbility_Jump

**Ability Properties:** `InstancedPerActor`, `NetExecutionPolicy = LocalPredicted`

### Activation Check

```cpp
bool ULyraGameplayAbility_Jump::CanActivateAbility(...)
{
    if (!Super::CanActivateAbility(...)) return false;
    
    ALyraCharacter* LyraCharacter = GetLyraCharacterFromActorInfo();
    return LyraCharacter && LyraCharacter->CanJump();
}
```

Delegates to `ALyraCharacter::CanJump()`, inherited from `ACharacter::CanJump()`, checks `bIsCrouched`, `GetCharacterMovement()->IsFalling()`, etc.

### Execution Flow

```
ActivateAbility()
  ├── CharacterJumpStart()
  │     └── ALyraCharacter::Jump() → ACharacter::Jump()
  │           └── CharacterMovement.SetJumping(true) + vertical initial velocity
  │
  └── AbilityTask_WaitInputRelease(bTestInitialState=true)
        └── OnRelease callback → CharacterJumpStop()
              └── ALyraCharacter::StopJumping() → ACharacter::StopJumping()
                    └── CharacterMovement.SetJumping(false)
```

`ACharacter::Jump()` / `StopJumping()` have built-in network synchronization capabilities, so GA_Jump does not need additional replication logic, relying on CharacterMovement replication.

### AbilityTask_WaitInputRelease

[UAbilityTask_WaitInputRelease](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/AbilitySystem/Abilities/LyraGameplayAbility_Jump.h) (custom version, not the standard UAbilityTask_WaitInputRelease):

- Registers callback to `AbilityReplicatedEventDelegate(InputReleased)`
- `bTestInitialState`: If true, immediately checks if input is already released on Activate; if so, directly triggers the callback
- Remote client uses `CallReplicatedEventDelegateIfSet` for predictive fallback synchronization

### AbilityTask_StartAbilityState (Deprecated)

Although `UAbilityTask_StartAbilityState` is marked `DEPRECATED`, GA_Jump still uses it for state lifecycle management. In `OnDestroy`:

```cpp
void UAbilityTask_StartAbilityState::OnDestroy(bool bInOwnerFinished)
{
    // Check if interrupted externally
    // Check if ended normally (bWasEnded || AbilityEnded)
    // Route to EndState or InterruptedState callback based on state
}
```

## GA_Dash (Root Motion Dash)

**Ability Properties:** `NetExecutionPolicy = LocalPredicted`

### Ability Configuration

Configured in Blueprint with the following parameters:
- Dash direction
- Dash distance / speed
- Dash animation (Montage)
- Root Motion force strength

### Execution Flow

```
ActivateAbility()
  ├── Compute dash direction (client/standalone only, server skips)
  │     └── Based on character orientation or movement input direction
  │
  ├── UAbilityTask_PlayMontageAndWait
  │     └── ASC::PlayMontage(AnimMontage, Rate, StartSection)
  │           └── Plays montage, syncs to RepAnimMontageInfo
  │
  └── UAbilityTask_ApplyRootMotionConstantForce
        └── MovementComponent->ApplyRootMotionSource(FRootMotionSource_ConstantForce)
              ├── WorldDirection → Computed direction
              ├── Strength → Dash speed
              ├── Duration → Dash duration
              ├── bIsAdditive → Whether additive
              ├── bUsesGravity → Whether affected by gravity
              └── FinishVelocityParams → End velocity behavior
```

### Network Sync Mechanism

**Montage Replication:**

```cpp
// ASC::PlayMontageInternal()
ASC->PlayMontageInternal(...)
{
    // Update RepAnimMontageInfo
    RepAnimMontageInfo.AnimMontage = MontageToPlay;
    RepAnimMontageInfo.SectionIdToPlay = 0;
    // ...
    ASC->ForceNetUpdate();  // Force network update
}
```

`FGameplayAbilityRepAnimMontage` / `RepAnimMontageInfo` is synced to clients via property replication. The client records montage data upon receipt for replay. Binds `OnPredictiveMontageRejected` callback on prediction key rejection.

**Root Motion Replication:**

Root Motion itself is replicated as a movement property through CharacterMovement and does not require additional synchronization code. The `LocalPredicted` strategy ensures the client predicts the dash position, with server validation and correction.

### Distance Calculation

GA_Dash's dash distance is not determined by animation-driven Root Motion, but calculated manually through `ApplyRootMotionConstantForce`'s `Strength * Duration`. This decouples the dash distance from the animation, making it easier to adjust and reuse.
