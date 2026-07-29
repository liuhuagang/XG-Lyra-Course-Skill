# ReplicationGraph

## Overview

ReplicationGraph is UE5's high-performance network replication framework. Lyra implements a three-layer node structure via `ULyraReplicationGraph` to optimize Actor network replication efficiency. The system routes Actors to different replication nodes based on their type, implementing strategies such as spatialization, frequency limiting, and always-relevant.

## File Structure

- [LyraReplicationGraph.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraReplicationGraph.h) / [LyraReplicationGraph.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraReplicationGraph.cpp) — Main graph and custom nodes
- [LyraReplicationGraphTypes.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraReplicationGraphTypes.h) — EClassRepNodeMapping enum definition
- [LyraReplicationGraphSettings.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraReplicationGraphSettings.h) / [LyraReplicationGraphSettings.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraReplicationGraphSettings.cpp) — Config class

## Three-Layer Node Structure

```
ULyraReplicationGraph
  ├── GridNode (UReplicationGraphNode_GridSpatialization2D)
  │     Handles Spatialize_Static / Dynamic / Dormancy Actors
  │
  ├── AlwaysRelevantNode (UReplicationGraphNode_ActorList)
  │     Handles RelevantAllConnections global always-relevant Actors
  │
  ├── ULyraReplicationGraphNode_PlayerStateFrequencyLimiter (Global node)
  │     Returns 2 PlayerStates per frame in a rolling manner
  │
  └── [Per Connection] ULyraReplicationGraphNode_AlwaysRelevant_ForConnection
          Connection-level always-relevant (Viewer, ViewTarget, Pawn, PlayerState, StreamingLevel Actors)
```

## EClassRepNodeMapping Actor Routing Strategy

[LyraReplicationGraphTypes.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraReplicationGraphTypes.h#L10-L20)

```cpp
enum class EClassRepNodeMapping : uint32
{
    NotRouted,                  // Not routed to any node, handled by special nodes
    RelevantAllConnections,     // Routed to AlwaysRelevantNode
    Spatialize_Static,          // Routed to GridNode: non-moving Actors
    Spatialize_Dynamic,         // Routed to GridNode: frequently moving Actors
    Spatialize_Dormancy,        // Routed to GridNode: static when dormant, dynamic when active
};
```

The routing strategy is managed via a `TClassMap<EClassRepNodeMapping>` mapping table, supporting three methods: hardcoded in code, custom via config, and runtime lazy initialization.

## RouteAddNetworkActorToNodes Routing Logic

[LyraReplicationGraph.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraReplicationGraph.cpp#L582-L623)

```cpp
void ULyraReplicationGraph::RouteAddNetworkActorToNodes(...)
{
    switch(Policy)
    {
        case NotRouted: break;
        case RelevantAllConnections:
            AlwaysRelevantNode->NotifyAddNetworkActor(ActorInfo);
            break;
        case Spatialize_Static:
            GridNode->AddActor_Static(ActorInfo, GlobalInfo);
            break;
        case Spatialize_Dynamic:
            GridNode->AddActor_Dynamic(ActorInfo, GlobalInfo);
            break;
        case Spatialize_Dormancy:
            GridNode->AddActor_Dormancy(ActorInfo, GlobalInfo);
            break;
    }
}
```

## Custom Nodes

### ULyraReplicationGraphNode_AlwaysRelevant_ForConnection

[LyraReplicationGraph.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraReplicationGraph.h#L67-L93)

Builds an always-relevant list for each connection every frame, does not maintain a persistent list. Contains:
- Viewers (InViewer + ViewTarget)
- PlayerController's Pawn
- PlayerState (50% frame throttling)
- Always-relevant Actors on StreamingLevel
- GameplayDebugger (in debug mode)

### ULyraReplicationGraphNode_PlayerStateFrequencyLimiter

[LyraReplicationGraph.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraReplicationGraph.h#L99-L123)

Returns at most `TargetActorsPerFrame (2)` PlayerStates per frame:
- `PrepareForReplication` rebuilds the list each frame, buckets all world PlayerStates by TargetActorsPerFrame
- `GatherActorListsForConnection` polls the corresponding bucket content based on `ReplicationFrameNum`

## Creation Flow

[LyraReplicationGraph.cpp](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraReplicationGraph.cpp#L135-L177)

Created via the `ConditionalCreateReplicationDriver` factory function, which is bound through `UReplicationDriver::CreateReplicationDriverDelegate()`.

```cpp
UReplicationDriver* ConditionalCreateReplicationDriver(UNetDriver* ForNetDriver, UWorld* World)
{
    const ULyraReplicationGraphSettings* LyraRepGraphSettings = GetDefault<ULyraReplicationGraphSettings>();
    if (LyraRepGraphSettings && LyraRepGraphSettings->bDisableReplicationGraph)
        return nullptr;  // Can disable ReplicationGraph, fall back to engine default

    TSubclassOf<ULyraReplicationGraph> GraphClass = ...;
    return NewObject<ULyraReplicationGraph>(GetTransientPackage(), GraphClass.Get());
}
```

## Configuration

[LyraReplicationGraphSettings.h](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/System/LyraReplicationGraphSettings.h)

- `bDisableReplicationGraph`: Whether to disable (default true)
- `DefaultReplicationGraphClass`: Configurable Graph subclass
- `TargetKBytesSecFastSharedPath`: FastShared path bandwidth
- `SpatialGridCellSize` / `SpatialBiasX/Y`: Spatial grid parameters
- `DynamicActorFrequencyBuckets`: Dynamic Actor frequency bucket count (default 3)
- `ClassSettings`: Custom per-class routing strategy via config
