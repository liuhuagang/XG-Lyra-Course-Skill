# GameplayMessageRouter

## Overview

Lyra's message system uses a two-layer architecture: the underlying `GameplayMessageRouter` plugin provides a generic Tag-based message routing mechanism, and the upper layer is the VerbMessage protocol layer built on top of it in `LyraGame/Messages/`. The message system supports server-to-client broadcast, local listening, Blueprint access, and chained message processor responses.

## Plugin Layer: GameplayMessageRouter

[GameplayMessageSubsystem.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Plugins/GameplayMessageRouter/Source/GameplayMessageRuntime/Public/GameFramework/GameplayMessageSubsystem.h)

Serves as a `UGameInstanceSubsystem`, providing a GameplayTag-based message publish-subscribe mechanism.

### Core Interface

- **BroadcastMessage**: Broadcasts messages by Tag, supports parent tag traversal (listeners on parent tags also receive child tag messages)
- **RegisterListener**: Registers listeners, configures listening parameters via `FGameplayMessageListenerParams`
- **K2_BroadcastMessage**: Blueprint version of the broadcast interface

### FGameplayMessageListenerParams

```cpp
FGameplayMessageListenerParams
{
    FGameplayTag Tag;                                     // Tag to listen for
    TFunction<void(FGameplayTag, const TSharedPtr<void>&)> OnMessageReceived;  // Callback
    EGameplayMessageMatch MatchType = EGameplayMessageMatch::ExactMatch;       // Match type
};
```

### Blueprint Support

Provides blueprint async nodes via `AsyncAction_ListenForGameplayMessage` for listening to messages in Blueprint.

## Application Layer: LyraGame/Messages/

[LyraGame/Messages/](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Messages/)

### FLyraVerbMessage Structure

[LyraVerbMessage.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Messages/LyraVerbMessage.h)

Core message structure, describing "who (Instigator) did what (Verb) to whom (Target)":

```cpp
USTRUCT(BlueprintType)
struct FLyraVerbMessage
{
    FGameplayTag Verb;                  // Action tag
    TObjectPtr<UObject> Instigator;     // Initiator
    TObjectPtr<UObject> Target;         // Target
    FGameplayTagContainer InstigatorTags;  // Initiator tags
    FGameplayTagContainer TargetTags;      // Target tags
    FGameplayTagContainer ContextTags;     // Context tags
    double Magnitude;                   // Magnitude value
};
```

### FLyraVerbMessageReplication

[LyraVerbMessageReplication.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Messages/LyraVerbMessageReplication.h) / [LyraVerbMessageReplication.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Messages/LyraVerbMessageReplication.cpp)

FFastArraySerializer-based server-to-client message replication mechanism:

1. Server calls `AddMessage()` to add a message to the `CurrentMessages` array, marks it dirty
2. FastArray auto-syncs to client via `NetDeltaSerialize`
3. Client re-broadcasts locally via `UGameplayMessageSubsystem` in `PostReplicatedAdd` / `PostReplicatedChange`
4. Local listeners receive the message

This design achieves **server-driven message replication + local message system distribution** decoupling.

### ULyraVerbMessageHelpers

[LyraVerbMessageHelpers.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Messages/LyraVerbMessageHelpers.h) / [LyraVerbMessageHelpers.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Messages/LyraVerbMessageHelpers.cpp)

- `GetPlayerStateFromObject`: Gets the corresponding PlayerState from any UObject
- `GetPlayerControllerFromObject`: Gets the corresponding PlayerController from any UObject
- `VerbMessageToCueParameters` / `CueParametersToVerbMessage`: Bidirectional conversion between VerbMessage and GameplayCueParameters

### FLyraNotificationMessage

[LyraNotificationMessage.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Messages/LyraNotificationMessage.h) / [LyraNotificationMessage.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Messages/LyraNotificationMessage.cpp)

Used for transient UI notifications such as kill feedback and pickup prompts, differentiated by `TargetChannel`.

```cpp
USTRUCT(BlueprintType)
struct FLyraNotificationMessage
{
    FGameplayTag TargetChannel;          // Target channel
    TObjectPtr<APlayerState> TargetPlayer;  // Target player
    FText PayloadMessage;                // Display text
    FGameplayTag PayloadTag;             // Payload tag
    TObjectPtr<UObject> PayloadObject;   // Payload object
};
```

### UGameplayMessageProcessor Base Class

[GameplayMessageProcessor.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Messages/GameplayMessageProcessor.h) / [GameplayMessageProcessor.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Messages/GameplayMessageProcessor.cpp)

Uses the **Template Method Pattern** as base class. Subclasses only need to override `StartListening()` and register message listeners within it; lifecycle is managed automatically.

- `BeginPlay()` → `StartListening()`
- `EndPlay()` → `StopListening()`, automatically cleans up all ListenerHandles
- `GetServerTime()`: Gets server time via `GetWorld()->GetGameState()->GetServerWorldTimeSeconds()`
