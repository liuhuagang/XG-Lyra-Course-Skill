# Game Module Code Supplement

## Overview

This chapter covers several scattered but important template code items in the LyraGame module not expanded in separate chapters, including PlayerController lifecycle fixes, VerbMessage replication integration in ALyraPlayerState, network mode utility enums, generic ProgressBar Widget, and simulated input Widget system.

## ALyraPlayerController::CleanupPlayerState

[LyraPlayerController.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Player/LyraPlayerController.cpp)

To solve the PlayerState resource leak issue in **Late Replication** scenarios, cleanup the associated PlayerState when PlayerController is destroyed:

```cpp
void ALyraPlayerController::CleanupPlayerState()
{
    // When PlayerController is destroyed, ensure PlayerState is also properly cleaned up
    if (PlayerState)
    {
        PlayerState->Destroy();
        PlayerState = nullptr;
    }
}
```

Typical scenario: When a client connects to a server, the PlayerState may arrive later than the PlayerController due to network latency, causing state desync or leaks.

## FLyraVerbMessageReplication Integration in ALyraPlayerState

[LyraVerbMessageReplication.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Messages/LyraVerbMessageReplication.h) / [LyraVerbMessageReplication.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Messages/LyraVerbMessageReplication.cpp)

FFastArraySerializer-based message replication container, typically used as a member of `ALyraPlayerState` or `ALyraGameState` to reliably sync VerbMessages from server to client.

By embedding `FLyraVerbMessageReplication` in PlayerState, the server-side call to `AddMessage()` is automatically synced to all clients via FastArray's `NetDeltaSerialize`.

Replication flow:
1. Server calls `AddMessage()` to add a message to the `CurrentMessages` array
2. FastArray incrementally syncs to client via `NetDeltaSerialize`
3. Client re-broadcasts locally via `UGameplayMessageSubsystem` in `PostReplicatedAdd` / `PostReplicatedChange`
4. All local listeners receive the message

## EBlueprintExposedNetMode & SwitchOnNetMode

[LyraPlayerController.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Player/LyraPlayerController.cpp)

UE5's new Blueprint-accessible network mode enum, used with the `SwitchOnNetMode` Blueprint node to execute different logic branches based on current network mode (Client/Server/Standalone/DedicatedServer).

```cpp
UENUM(BlueprintType)
enum EBlueprintExposedNetMode : int
{
    NetMode_Client,
    NetMode_Standalone,
    NetMode_ListenServer,
    NetMode_DedicatedServer,
};
```

## UMaterialProgressBar

Generic material-driven progress bar Widget, displays progress by updating material parameters (scalar/vector).

### Core Mechanism

- Uses MID (Material Instance Dynamic) as the progress visualization foundation
- Supports two animation modes: `AnimateProgressFromStart` (animate from start position) and `AnimateProgressFromCurrent` (continue from current position)
- Driven by material parameters: ScalarParameterValues and VectorParameterValues

### Use Cases

Display values requiring smooth transitions (health, progress, etc.) in UI. Uses material parameters instead of Slate drawing for performance optimization and visual consistency.

## ULyraSimulatedInputWidget

[LyraSimulatedInputWidget.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/UI/)

Simulated input Widget system for touch/mobile manual input.

### Core Classes

- **ULyraSimulatedInputWidget**: Base simulated input Widget, provides `InputKeyValue` (single key value) and `InputKeyValue2D` (2D axis value) input types. `QueryKeyToSimulate` method queries the key to simulate.
- **ULyraJoystickWidget**: Touch joystick Widget, handles touch drag events via `HandleTouchDelta`; `StickRange` defines the joystick's max offset range.
- **ULyraTouchRegion**: Touch region Widget, `bShouldSimulateInput` controls whether input simulation is enabled.

### Application Scenario

On mobile or touch devices, these Widgets map touch input to in-game key/axis inputs, enabling non-physical controller operation simulation.
