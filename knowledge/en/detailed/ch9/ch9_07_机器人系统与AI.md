# Bot System & AI

## Architecture Overview

The bot system is responsible for spawning AI-controlled players (Bots), assigning them to teams, and driving them to participate in the game. AI behavior is based on behavior trees and GAS integration.

## Bot Creation

[ULyraBotCreationComponent](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/)

```cpp
UCLASS()
class ULyraBotCreationComponent : public UGameStateComponent
{
    // Number of Bots to create
    UPROPERTY(EditDefaultsOnly)
    int32 NumBotsToCreate = 5;

    // Bot's Pawn class
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<APawn> BotPawnClass;

    // Create Bots
    void ServerCreateBots();
};
```

### Bot Creation Flow

1. `ServerCreateBots` is called after Experience loads
2. Server spawns `ALyraPlayerBotController`
3. Registers Bot Controller with the team system
4. Subsequently, the spawn manager (`ULyraPlayerSpawningManagerComponent`) assigns spawn points

## Bot Controller

[ALyraPlayerBotController](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/)

```cpp
UCLASS()
class ALyraPlayerBotController : public AModularAIController, public ILyraTeamAgentInterface
{
    // Team-related
    virtual void OnPossess(APawn* InPawn) override;
    virtual void OnUnPossess() override;

    // Team ID management
    UPROPERTY()
    FOnLyraTeamIndexChangedDelegate OnTeamIndexChanged;
    int32 MyTeamID;

    // Restart
    void ServerRestartController();
};
```

- Inherits `AModularAIController` for modular component support
- Implements `ILyraTeamAgentInterface` to participate in the team system
- On death, respawns via `ServerRestartController`

### Bot Cheats

[ULyraBotCheats](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/code/Lyra/Source/LyraGame/)

- Console commands: spawn/destroy Bots
- Toggle Bot behavior

## Behavior Tree & Environment Query

### Behavior Tree

- Defines Bot behavior logic: patrol, search, combat, chase
- Stores targets, locations, and states via Blackboard
- AI perception system drives behavior tree nodes
- Bots use visual perception to detect enemies and auditory perception to detect events

### Environment Query (EQS)

- Selects best shooting positions, cover positions, movement targets
- Test conditions: visibility, distance, team relationship

### GAS Integration

- Bots use GAS to activate abilities and respond to GameplayEvents
- Bots have the same AbilitySet as human players
- Ability activation and cooldowns are handled through `ULyraAbilitySystemComponent`
- Bot behavior tree nodes call GAS-related functionality

### LyraShooterBot Controller

Flow:
1. Spawn `ALyraPlayerBotController`
2. Initialize team ID
3. Spawn Pawn → Possess → Apply Experience's AbilitySet
4. Start behavior tree
5. Death → wait for respawn → re-Possess
