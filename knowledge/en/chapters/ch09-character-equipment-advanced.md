# Chapter 9: Character & Equipment Advanced

## Overview

Chapter 9 delves into the advanced mechanisms of Lyra's character system, covering the cosmetic system, team system, screen indicators, async loading, accolades and kill records, game phase management, bot AI, and silhouette outlines.

## Knowledge Points

### [Cosmetic System](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch9/ch9_01_换装系统.md)
- Adding and removing character parts (FLyraCharacterPart)
- FastArray (FLyraCharacterPartList) network synchronization
- Animation layer selection (FLyraAnimLayerSelectionEntry)
- Dual-component collaboration between Controller and Pawn
- Cosmetic cheats and developer settings

### [Team System](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch9/ch9_02_队伍系统.md)
- Team interface ILyraTeamAgentInterface (extends IGenericTeamAgentInterface)
- Team display asset ULyraTeamDisplayAsset
- Team info hierarchy: ALyraTeamInfoBase / PublicInfo / PrivateInfo
- Team subsystem ULyraTeamSubsystem
- Team creation component ULyraTeamCreationComponent (Experience-driven)
- Player spawning ULyraPlayerSpawningManagerComponent
- TDM mode battlefield spawning strategy (UTDM_PlayerSpawningManagmentComponent)
- Async team observation (UAsyncAction_ObserveTeam / ObserveTeamColors)

### [Indicator System](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch9/ch9_03_指示器系统.md)
- Indicator descriptor UIndicatorDescriptor (data/projection/layout)
- Indicator manager ULyraIndicatorManagerComponent
- Widget interface IIndicatorWidgetInterface
- UMG layer UIndicatorLayer → Slate canvas SActorCanvas
- 5 projection modes (ComponentPoint / BoundingBox / ScreenBoundingBox / ActorBoundingBox / ActorScreenBoundingBox)
- Edge snapping and arrow indicators
- Object pool reuse
- Interaction system integration

### [AsyncMixin Async Loading](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch9/ch9_04_AsyncMixin异步加载.md)
- FAsyncMixin's non-copyable design
- Chained loading sequence (AsyncLoad → AsyncCondition → AsyncEvent)
- FLoadingState step management
- FAsyncCondition ticker polling (0.16s)
- Zero memory overhead: only allocated when used
- FAsyncScope standalone usage wrapper

### [Accolades & Kill Records](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch9/ch9_05_荣誉与击杀记录.md)
- Accolade definition (FLyraAccoladeDefinitionRow, DataRegistry)
- Notification message (FLyraNotificationMessage)
- Accolade broadcast Widget (ULyraAccoladeHostWidget, FAsyncMixin)
- VerbMessage system (FLyraVerbMessage, UGameplayMessageSubsystem)
- Kill message flow: Death → Eliminate.Message → multiple processors
- UAssistProcessor: damage recording + assist broadcast
- UElimChainProcessor: kill chain detection (time window 4.5s)
- UElimStreakProcessor: streak detection
- Client broadcast (ALyraPlayerState::ClientBroadcastMessage)
- Kill Feed UI (ListView)
- Score processing (TeamInfo/PlayerState TagStack)

### [Game Phase System](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch9/ch9_06_游戏阶段系统.md)
- Phase ability ULyraGamePhaseAbility (bound to GameplayTag)
- Phase subsystem ULyraGamePhaseSubsystem (WorldSubsystem)
- Phase mutual exclusion logic (parent phase survives vs sibling replacement vs same-level replacement)
- Phase observers (WhenPhaseStartsOrIsActive / WhenPhaseEnds)
- Warm-up phase (invulnerability + countdown)
- Combat phase (reset + scene cleanup)
- Scoring phase (global invulnerability + PlayNextGame)
- Development tool ULyraDevelopmentStatics (skip warm-up in PIE)
- Scoreboard UI

### [Bot System & AI](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch9/ch9_07_机器人系统与AI.md)
- Bot creation component ULyraBotCreationComponent (Experience-driven)
- Bot controller ALyraPlayerBotController (AModularAIController + ILyraTeamAgentInterface)
- Bot cheats ULyraBotCheats
- Behavior Tree + EQS + GAS integration
- Bot respawn flow

### [Silhouette Outline](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch9/ch9_08_外轮廓线.md)
- Stencil buffer approach
- Custom Depth Stencil Value
- Post-process material sampling + edge detection
- Team color switching

## Lecture List

| ID | Title | File |
|----|-------|------|
| 100 | Cosmetic System | UE5_Lyra学习指南_100_换装系统.md |
| 101 | Team System | UE5_Lyra学习指南_101_队伍系统.md |
| 102 | Silhouette Outline | UE5_Lyra学习指南_102_外轮廓线.md |
| 103 | Indicator System | UE5_Lyra学习指南_103_指示器系统.md |
| 104 | AsyncMixin | UE5_Lyra学习指南_104_异步混入AsyncMixin.md |
| 105 | Accolade System | UE5_Lyra学习指南_105_荣誉系统.md |
| 106 | Kill Records | UE5_Lyra学习指南_106_击杀记录.md |
| 107 | Game Phase System | UE5_Lyra学习指南_107_游戏阶段系统.md |
| 108 | Bot System | UE5_Lyra学习指南_108_机器人系统.md |
| 109 | Behavior Tree & Environment Query | UE5_Lyra学习指南_109_行为树和环境查询.md |
