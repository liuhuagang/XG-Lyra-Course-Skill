# Chapter 10: Game Flow & Systems

## Overview

Chapter 10 covers advanced systems and engineering tools related to game flow in the Lyra project, including the cheat debug system, message routing, context effects, ReplicationGraph network optimization, automated testing pipeline, replay system, editor extensions, Pocket plugin, and the complete Bomberman gameplay case study.

## Knowledge Points

### [Cheat Debug System](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch10/ch10_01_作弊调试系统.md)
- ULyraCheatManager built-in cheat commands (God / camera debug / damage heal)
- CheatManagerExtension auto-registration mode (BotCheats / CosmeticCheats / TeamCheats)
- ULyraGameplayRpcRegistrationComponent HTTP RPC remote cheats

### [GameplayMessageRouter](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch10/ch10_02_GameplayMessageRouter消息路由.md)
- Plugin layer: UGameplayMessageSubsystem (UGameInstanceSubsystem)
- Tag-based pub/sub message mechanism, parent Tag traversal
- Application layer: FLyraVerbMessage (Instigator / Verb / Target / Magnitude)
- FLyraVerbMessageReplication (FFastArraySerializer server→client sync + local re-broadcast)
- FLyraNotificationMessage (UI notification message)
- UGameplayMessageProcessor template method base class

### [Context Effects System](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch10/ch10_03_足迹上下文系统.md)
- ILyraContextEffectsInterface (BlueprintNativeEvent)
- UAnimNotify_LyraContextEffects (optional LineTrace + interface discovery)
- ULyraContextEffectsLibrary (EffectTag + Context → Sound/Niagara mapping)
- ULyraContextEffectComponent (triple context aggregation + physical surface conversion)
- ULyraContextEffectsSubsystem (WorldSubsystem + ActiveActorEffectsMap)
- ULyraContextEffectsSettings (DefaultGame.ini SurfaceType → GameplayTag mapping)

### [ReplicationGraph](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch10/ch10_04_ReplicationGraph网络复制图.md)
- EClassRepNodeMapping (NotRouted / RelevantAllConnections / Spatialize_Static / Dynamic / Dormancy)
- ULyraReplicationGraph three-layer node structure (GridNode / AlwaysRelevantNode / PlayerStateFrequencyLimiter)
- RouteAddNetworkActorToNodes dispatch routing
- ConditionalCreateReplicationDriver factory
- ULyraReplicationGraphSettings (bDisableReplicationGraph / grid parameters / frequency buckets)

### [Automated Test System](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch10/ch10_05_自动化测试系统.md)
- CQTest framework (TEST_CLASS_WITH_FLAGS / BEFORE_EACH / TEST_METHOD)
- FMapTestSpawner / ACTOR_ANIMATION_TEST / ACTOR_ANIMATION_NETWORK_TEST
- ShooterTests plugin input simulation helpers
- Device property tests, async message tests
- Gauntlet framework: ULyraTestControllerBootTest (20s placeholder)

### [Replay System](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch10/ch10_06_回放系统.md)
- ULyraReplaySubsystem (UGameInstanceSubsystem): PlayReplay / RecordClientReplay / CleanupLocalReplays / SeekInActiveReplay
- UAsyncAction_QueryReplays (Blueprint async node + EnumerateStreams)
- ULyraReplayListEntry (FNetworkReplayStreamInfo wrapper)
- ALyraReplayPlayerController (OnPawnSet follows recorder)
- ReplaySpectatorPlayerControllerClass GameMode configuration

### [Game Template Code Supplement](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch10/ch10_07_Game模板代码补充.md)
- ALyraPlayerController::CleanupPlayerState delayed replication fix
- FLyraVerbMessageReplication FastArraySerializer
- EBlueprintExposedNetMode / SwitchOnNetMode
- UMaterialProgressBar (MID-driven animation)
- Touch input simulation (ULyraSimulatedInputWidget / ULyraJoystickWidget / ULyraTouchRegion)

### [Editor Module Code](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch10/ch10_08_Editor模块代码.md)
- FLyraEditorModule (GameplayCue editor delegates / PIE events / asset type registration / toolbar)
- FGameEditorStyle (Slate style set)
- ULyraEditorEngine (FirstTickSetup / PreCreatePIEInstances ForceStandaloneNetMode)
- UEditorValidator hierarchy (SourceControl / Blueprints / Load / MaterialFunctions)
- UContentValidationCommandlet content validation pipeline
- Utility functions (CheckChaosMeshCollision / CreateRedirectorPackage, etc.)

### [Pocket Plugin Supplement](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch10/ch10_09_Pocket插件补充.md)
- Pocket level streaming: UPocketLevel (DataAsset) / UPocketLevelSubsystem (WorldSubsystem) / UPocketLevelInstance
- Pocket screenshot system: UPocketCaptureSubsystem / UPocketCapture (SceneCapture2D wrapper)
- Two-frame Mip streaming strategy / Triple render target / ShowFlags optimization

### [Bomberman Top-Down Arena](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch10/ch10_10_炸弹人玩法俯视角竞技场.md)
- UTopDownArenaAttributeSet (BombsRemaining / BombCapacity / BombRange / MovementSpeed)
- UTopDownArenaMovementComponent (attribute-driven movement speed)
- ULyraCameraMode_TopDownArenaCamera (fixed top-down view)
- Bomb mechanics (Blueprint: place / burn / recursive explosion / chain)
- Pickup system, Experience configuration, UI extension points

### [Gyra Behavior Tree Supplement](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch10/ch10_11_Gyra行为树补充.md)
- Minion chase / Boss phase behavior trees
- BTDecorator_HealthCheck / BTDecorator_CheckDistance
- BTT_Lich_Transport / BTT_SetMovementSpeed / BTT_Monster_MeleeAttack
- Navigation Mesh jump links

## Lecture List

| ID | Title | File |
|----|-------|------|
| 110 | Gyra Behavior Tree Supplement | UE5_Lyra学习指南_110_Gyra行为树补充.docx |
| 111 | Cheat Debug System | UE5_Lyra学习指南_111_作弊调试系统.md |
| 112 | GameplayMessageRouter | UE5_Lyra学习指南_112_GameplayMessageRouter.md |
| 113 | Context Effects System | UE5_Lyra学习指南_113_足迹上下文系统.md |
| 114 | ReplicationGraph | UE5_Lyra学习指南_114_ReplicationGraph.md |
| 115 | Automated Test System | UE5_Lyra学习指南_115_自动化测试系统.md |
| 116 | Gauntlet Automation Framework | UE5_Lyra学习指南_116_Gauntlet自动化框架.md |
| 117 | Replay System | UE5_Lyra学习指南_117_回放系统.md |
| 118 | Game Template Code | UE5_Lyra学习指南_118_Game模板代码.md |
| 119 | Editor Module Code | UE5_Lyra学习指南_119_Editor模块代码.md |
| 119_泽 | Set Deduplication Code Supplement | UE5_Lyra学习指南_119_泽_集合去重代码.md |
| 120 | Pocket Plugin Supplement | UE5_Lyra学习指南_120_Pocket插件补充.md |
| 121 | Bomberman Gameplay | UE5_Lyra学习指南_121_炸弹人玩法.md |
| 122 | Final Chapter | UE5_Lyra学习指南_122_终章.md |
