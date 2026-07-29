# Player Spawning System

## Overview

Lyra's player spawning system is centrally managed by `ULyraPlayerSpawningManagerComponent`, which replaces the traditional `AGameModeBase::ChoosePlayerStart` and `FindPlayerStart` methods.

## ULyraPlayerSpawningManagerComponent

[ULyraPlayerSpawningManagerComponent](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/Player/LyraPlayerSpawningManagerComponent.h) inherits from `UGameStateComponent` and is attached to `ALyraGameState`.

### Responsibilities

1. **Manage all player spawn points** — Collect `APlayerStart` and custom start points in the scene
2. **Assign spawn points** — Implement team spawning logic, avoid conflicts
3. **Handle respawning** — Manage respawn flow after player death

### Core Logic

```cpp
UCLASS()
class ULyraPlayerSpawningManagerComponent : public UGameStateComponent
{
    GENERATED_BODY()

public:
    // Get player start point
    virtual AActor* OnChoosePlayerStart(AController* Player, TArray<APlayerStart*>& Starts);

    // Get the best player start point
    APlayerStart* GetBestPlayerStart(AController* Player);

    // Initialize spawn points
    virtual void InitializeStarts();

protected:
    // Cache of all player start points
    TArray<APlayerStart*> PlayerStarts;

    // Occupied spawn points
    TSet<APlayerStart*> OccupiedStarts;
};
```

## Player Spawn Points

### UE Built-in APlayerStart

`APlayerStart` is UE's built-in player start Actor, containing:
- `PlayerStartTag` — Used to tag spawn points for specific teams
- Location and rotation — Position and orientation at spawn

### Lyra's Extensions

Lyra uses custom player spawn points that may include:
- Team affiliation markers
- Priority settings
- Special spawn rules (e.g., random, farthest, etc.)

## Spawn Flow

```
ALyraGameMode::HandleStartingNewPlayer
  → ULyraPlayerSpawningManagerComponent::OnChoosePlayerStart
    → Select appropriate start point from PlayerStarts
    → Consider: team affiliation, occupancy, priority
  → SpawnDefaultPawnAtTransform
    → Create ALyraPawn
    → Initialize PawnExtensionComponent
  → Possess Pawn
    → PlayerController controls Pawn
```

## Team Spawning Logic

For team-based modes, players from different teams should spawn at different locations:

- Filter start points matching the team tag from `PlayerStarts`
- Start points for the same team are arranged in zones
- Avoid spawn points from different teams being too close together
