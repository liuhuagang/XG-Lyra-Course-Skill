# Lyra Deep Dive - Course Syllabus

> Course knowledge system index. Compiled based on course handouts (docs/), LyraStarterGame source code (code/).
> For future queries about course structure, reference this file directly without re-reading handouts.

---

## I. Course Overview

| Dimension | Data |
|-----------|------|
| Course Name | Lyra Deep Dive |
| Platform | Bilibili Classroom |
| Instructor | Unreal Xiaogang |
| Total Handouts | **122** documents (numbered 001~122, in .md and .docx formats) |
| Source Code | **LyraStarterGame** (Epic Games official sample project, UE5) |
| Source Code Modules | **26** C++ subdirectories, **23** public dependency modules |
| Prerequisite Course | [UEC++ From Basics to Advanced](https://www.bilibili.com/cheese/play/ss28043) |
| Coverage | Project setup, Gameplay framework, Settings system, UI, Character & Input, GAS, Equipment & Weapons, Cosmetics, Teams, AI, Replays, Editor extensions, etc. |

---

## II. Knowledge Architecture

Organized by **subsystem** (not teaching order), each chapter corresponds to an independent knowledge entry file, with granular documents in `ch{N}/` subdirectories.

| Ch | Name | Handout Numbers | Source Directory | Core Systems |
|----|------|-----------------|-----------------|-------------|
| 1 | [Project Architecture & Setup](#chapter-1-project-architecture--setup) | 001~009 | System/ | Engine, GameInstance, AssetManager, Logging, Target/Build |
| 2 | [Experience & Gameplay Framework](#chapter-2-experience--gameplay-framework) | 010~027 | GameModes/, Player/, System/ | Experience, GM/GS/PS/PC, LocalPlayer |
| 3 | [Settings System](#chapter-3-settings-system) | 028~038 | Settings/, Audio/, Hotfix/ | Local/Shared Settings, Audio, SaveGame, GameFeature, Hotfix |
| 4 | [UI Architecture & Login Flow](#chapter-4-ui-architecture--login-flow) | 039~054 | UI/, GameModes/ | LoadingScreen, Login, HUD, Sessions, CommonUser |
| 5 | [Settings UI Implementation](#chapter-5-settings-ui-implementation) | 055~064 | Settings/ (Registry), Settings/CustomSettings/ | GameSettingRegistry, Platform Settings, Music |
| 6 | [Character & Input System](#chapter-6-character--input-system) | 065~072, 078~079 | Character/, Input/, Camera/, Animation/ | Character base, PawnExtension, HeroComponent, Input, Camera |
| 7 | [GAS Ability System](#chapter-7-gas-ability-system) | 080~090 | AbilitySystem/, Interaction/, Inventory/ | ASC/GA, AttributeSet, GEEC, Health, Inventory, Interaction |
| 8 | [Equipment & Weapon System](#chapter-8-equipment--weapon-system) | 091~099 | Equipment/, Weapons/, Feedback/ | Equipment management, Ranged weapons, Grenades, Assisted aiming |
| 9 | [Cosmetics, Teams & Indicators](#chapter-9-cosmetics-teams--indicators) | 100~109 | Cosmetics/, Teams/, UI/IndicatorSystem/, GameModes/, Development/, Messages/ | Cosmetics, Teams, Indicators, AsyncMixin, Accolades, Kills, GamePhase, Bots, Stencil |
| 10 | [Game Flow & Systems](#chapter-10-game-flow--systems) | 110~122 | Development/, Replays/, Editor/, PocketWorlds/, TopDownArena/ | Cheats & Debug, MessageRouter, ContextEffects, ReplicationGraph, Automated Tests, Replays, Editor, Pocket, Bomberman |

---

## III. Chapter Details

### Chapter 1 Project Architecture & Setup

| Item | Description |
|------|-------------|
| Handout Range | `UE5_Lyra学习指南_001` ~ `009` |
| Source Directory | System/ (LyraGameEngine, LyraGameInstance, LyraAssetManager) |
| Core Concepts | Project introduction, Source code overview, Creating the project, Target/Build files, Engine class, Log categories, Importing assets |
| Supplements | `_补_01` SPathView plugin display, `_泽_` series supplementary knowledge |
| Knowledge Entry | [ch1/ knowledge files](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch1/) — 8 knowledge files covering project overview, Engine/Instance, AssetManager, Logging, Target/Build system |

**Key Classes**: ULyraGameEngine, ULyraGameInstance, ULyraAssetManager, ULyraLogChannels

### Chapter 2 Experience & Gameplay Framework

| Item | Description |
|------|-------------|
| Handout Range | `010` ~ `027` |
| Source Directory | GameModes/, Player/, System/ (partial) |
| Core Concepts | Experience definition and management, GameMode/GameState/PlayerState/PlayerController/LocalPlayer workflow, Player spawning, Developer settings, Platform emulation |
| Key Mechanisms | Experience async loading flow, GameFeature plugin activation, PawnData management |
| Knowledge Entry | [ch2/ knowledge files](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch2/) — 8 knowledge files covering Experience definition and loading, GM/GS/PS/PC, Player spawning, Developer settings |

**Key Classes**: ULyraExperienceDefinition, ULyraExperienceManager, ULyraExperienceManagerComponent, ULyraGameMode, ULyraGameState, ULyraPlayerState, ULyraPlayerController, ULyraLocalPlayer, ULyraPlayerSpawningManagerComponent

### Chapter 3 Settings System

| Item | Description |
|------|-------------|
| Handout Range | `028` ~ `038` |
| Source Directory | Settings/, Audio/, Hotfix/, GameFeatures/ |
| Core Concepts | GameUserSettings, Local/Shared game settings, Audio mixing, Platform rendering settings, SaveGame, LyraGameFeature, Hotfix, GameplayCueManager |
| Key Mechanisms | Settings layered architecture (Local vs Shared), GameFeature plugin dynamic loading |
| Knowledge Entry | [ch3/ knowledge files](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch3/) — 7 knowledge files covering Local/Shared settings, Audio mixing, SaveGame, Hotfix, GameFeature |

**Key Classes**: ULyraSettingsLocal, ULyraSettingsShared, ULyraAudioMixEffectsSubsystem, ULyraAudioSettings, ULyraHotfixManager, UGameFeatureAction, ULyraGameFeaturePolicy

### Chapter 4 UI Architecture & Login Flow

| Item | Description |
|------|-------------|
| Handout Range | `039` ~ `054` |
| Source Directory | UI/, UI/Foundation/, UI/Frontend/, UI/Common/ |
| Core Concepts | CommonLoadingScreen, Login flow, HUD Layout, Button system, Dialogs, Session system, CommonUser, Tab lists |
| Key Mechanisms | Async UI push, Strategy loading, CommonUI's three modes: Layout/Push/Show |
| Knowledge Entry | [ch4/ knowledge files](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch4/) — 9 knowledge files covering HUD architecture, Buttons/Dialogs/Session system, CommonUser, Tab lists, Settings UI |

**Key Classes**: ULyraLoadingScreenSubsystem, ULyraHUD, ULyraHUDLayout, ULyraButtonBase, ULyraTabListWidgetBase, ULyraFrontendStateComponent

### Chapter 5 Settings UI Implementation

| Item | Description |
|------|-------------|
| Handout Range | `055` ~ `064` |
| Source Directory | Settings/ (GameSettingRegistry), Settings/CustomSettings/, Settings/Screens/ |
| Core Concepts | GameSettingRegistry system, Audio/Controller/KeyboardMouse/Video/Gameplay settings, Performance analysis, Music component |
| Key Mechanisms | Custom SettingValue types (Resolution, Language, Quality, etc.) |
| Knowledge Entry | [ch5/ knowledge files](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch5/) — 8 knowledge files covering GameSettingRegistry system, Audio/Video/Input/Gameplay/Performance settings, Music component |

**Key Classes**: ULyraGameSettingRegistry, ULyraSettingValueDiscrete_Resolution, ULyraSettingKeyboardInput, ULyraSettingAction_SafeZoneEditor

### Chapter 6 Character & Input System

| Item | Description |
|------|-------------|
| Handout Range | `065` ~ `072`, `078` ~ `079` |
| Source Directory | Character/, Input/, Camera/, Animation/ |
| Core Concepts | Character base class, Movement component and Animation Blueprint, PawnExtension hub component, HeroComponent, Input system, Camera modes, Third-person penetration |
| Key Mechanisms | InitState initialization state machine, Enhanced Input configuration, CameraMode blending |
| Knowledge Entry | [ch6/ knowledge files](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch6/) — 13 knowledge files + 5 prompt files covering Character hierarchy, PawnExtension, HeroComponent, Input binding, Camera modes, Animation Blueprint |

**Key Classes**: ALyraCharacter, ALyraPawn, ULyraPawnExtensionComponent, ULyraHeroComponent, ULyraCharacterMovementComponent, ULyraInputComponent, ULyraInputConfig, ULyraCameraComponent, ULyraCameraMode

### Chapter 7 GAS Ability System

| Item | Description |
|------|-------------|
| Handout Range | `080` ~ `090` |
| Source Directory | AbilitySystem/, Interaction/, Inventory/ |
| Core Concepts | ASC and GA architecture, AbilitySet, GA_Jump/GA_Dash, AttributeSet, GEEC, Health component, Inventory system, Interaction system |
| Key Mechanisms | AbilityTagRelationship, GameplayCue management, AbilityCost, Inventory Fragment pattern |
| Knowledge Entry | [ch7/01~09 knowledge files](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch7/) |

**Key Classes**: ULyraAbilitySystemComponent, ULyraGameplayAbility, ULyraAbilitySet, ULyraHealthSet, ULyraCombatSet, ULyraDamageExecution, ULyraGameplayCueManager, ULyraInventoryManagerComponent, ULyraInventoryItemDefinition, ULyraGameplayAbility_Interact

### Chapter 8 Equipment & Weapon System

| Item | Description |
|------|-------------|
| Handout Range | `091` ~ `099` |
| Source Directory | Equipment/, Weapons/, Feedback/, UI/Weapons/ |
| Core Concepts | Equipment system architecture, Ranged weapon definition, Fire ability, Weapon crosshair, Damage feedback, Damage numbers, Weapon spawner, Grenades, Assisted aiming |
| Key Mechanisms | EquipmentInstance/Definition/Fragment pattern, Spread/Heat system, Two-stage bullet trace, Prediction system |
| Knowledge Entry | [ch8/ knowledge files](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch8/) |

**Key Classes**: ULyraEquipmentManagerComponent, ULyraEquipmentDefinition, ULyraEquipmentInstance, ULyraQuickBarComponent, ULyraRangedWeaponInstance, ULyraGameplayAbility_RangedWeapon, ULyraWeaponInstance, ULyraWeaponStateComponent, LyraContextEffectComponent, ULyraWeaponSpawner, LyraNumberPopComponent_NiagaraText

### Chapter 9 Cosmetics, Teams & Indicators

| Item | Description |
|------|-------------|
| Handout Range | `100` ~ `109` |
| Source Directory | Cosmetics/, Teams/, UI/IndicatorSystem/, GameModes/ (GamePhase), Messages/, Development/ |
| Core Concepts | Cosmetics system (Character parts), Teams system, Stencil outline, Indicator system (Nameplates/Arrows), AsyncMixin async loading, Accolade system, Kill feed, GamePhase management, Bot AI, Behavior Tree and EQS |
| Key Mechanisms | CharacterPart definition and network sync, TeamColor/Stencil rendering, Canvas-based indicators, AsyncMixin chained loading, VerbMessage routing, GamePhase state machine |
| Knowledge Entry | [ch9/01~08 knowledge files](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch9/) |

**Key Classes**: ULyraPawnComponent_CharacterParts, ULyraControllerComponent_CharacterParts, ULyraTeamSubsystem, ULyraTeamPublicInfo, ULyraTeamDisplayAsset, ULyraIndicatorManagerComponent, UIndicatorDescriptor, FAsyncMixin, ULyraGamePhaseSubsystem, ULyraGamePhaseAbility, ULyraBotCreationComponent, ALyraPlayerBotController

### Chapter 10 Game Flow & Systems

| Item | Description |
|------|-------------|
| Handout Range | `110` ~ `122` |
| Source Directory | Development/ (Cheats & Debug), Replays/, Editor/, ContextEffects/, Plugins/GameplayMessageRouter/, Plugins/PocketWorlds/, Plugins/GameFeatures/TopDownArena/ |
| Core Concepts | Cheats & Debug, GameplayMessageRouter, Context effects, ReplicationGraph, Automated testing (CQTest + Gauntlet), Replay system, Editor module, Pocket plugin, Bomberman gameplay |
| Key Mechanisms | Network Replication Graph optimization, Automated test pipeline, Pocket world streaming and screenshots |
| Knowledge Entry | [Chapter 10 - Game Flow & Systems](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/第十章-游戏流程与系统.md) |

**Key Classes**: ULyraCheatManager, ULyraGameplayRpcRegistrationComponent, UGameplayMessageSubsystem, FLyraVerbMessage, ULyraContextEffectComponent, ULyraReplicationGraph, ULyraTestControllerBootTest, ULyraReplaySubsystem, ALyraReplayPlayerController, FLyraEditorModule, UPocketLevelSubsystem, UPocketCaptureSubsystem, UTopDownArenaAttributeSet

**Knowledge Files**: [ch10/01~11](file:///d:/UPS/GitLab/XGUnrealNote/Courses/Lyra-Deep-Dive/knowledge/ch10/) — Cheat & Debug system, GameplayMessageRouter, Context Effects system, ReplicationGraph, Automated test system, Replay system, Game template code supplement, Editor module code, Pocket plugin supplement, Bomberman gameplay, Gyra behavior tree supplement

---

## IV. Complete Handout Number Mapping

| Number Range | Knowledge Chapter | Description |
|-------------|-------------------|-------------|
| 001~009 | Chapter 1 | Project Architecture & Setup |
| 010~027 | Chapter 2 | Experience & Gameplay Framework |
| 028~038 | Chapter 3 | Settings System |
| 039~054 | Chapter 4 | UI Architecture & Login Flow |
| 055~064 | Chapter 5 | Settings UI Implementation |
| 065~072 | Chapter 6 | Character & Input System |
| 078~079 | Chapter 6 | Camera System |
| 080~090 | Chapter 7 | GAS Ability System |
| 091~099 | Chapter 8 | Equipment & Weapon System |
| 100~109 | Chapter 9 | Cosmetics, Teams & Indicators |
| 110~122 | Chapter 10 | Game Flow & Systems |

> 038, 041, 072~077, 084, 109, 110 are in .docx format (extracted to .txt, text readable. Contains many editor screenshots, UI operation details require viewing the original file); `_泽_` and `_AI` suffixes indicate supplementary/topic content, filed under the corresponding chapter.

---

## V. Source Directory to Knowledge Chapter Mapping

| Source Directory | Corresponding Chapter | Key File Count |
|-----------------|----------------------|----------------|
| AbilitySystem/ | Chapter 7 | 10 + 14 sub-files |
| Animation/ | Chapter 6 | 1 |
| Audio/ | Chapter 3 | 2 |
| Camera/ | Chapter 6 | 7 |
| Character/ | Chapter 6 | 8 |
| Cosmetics/ | Chapter 9 | 11 |
| Development/ | Chapters 9, 10 | 3 |
| Equipment/ | Chapter 8 | 6 |
| Feedback/ | Chapter 8 | 10 |
| GameFeatures/ | Chapter 3 | 8 |
| GameModes/ | Chapters 2, 10 | 10 |
| Hotfix/ | Chapter 3 | 3 |
| Input/ | Chapter 6 | 6 |
| Interaction/ | Chapter 7 | 11 |
| Inventory/ | Chapter 7 | 8 |
| Messages/ | Chapter 10 | 5 |
| Performance/ | Chapters 5, 10 | 3 |
| Physics/ | — | 2 |
| Player/ | Chapter 2 | 8 |
| Replays/ | Chapter 10 | 2 |
| Settings/ | Chapters 3, 5 | 14 |
| System/ | Chapters 1, 2 | 14 |
| Teams/ | Chapter 9 | 11 |
| Tests/ | Chapter 10 | 2 |
| UI/ | Chapters 4, 5, 8, 9 | 30+ |
| Weapons/ | Chapter 8 | 8 |

---

## VI. Dependency Graph

```
Engine Layer (Chapter 1)
  └─ AssetManager / GameInstance
       └─ Experience Framework (Chapter 2)
            ├─ GameMode / GameState / PlayerState / PC / LP (Chapter 2)
            ├─ Settings System (Chapter 3)
            │    ├─ Local/Shared Settings
            │    ├─ Audio / SaveGame
            │    └─ GameFeature / Hotfix
            ├─ UI Architecture (Chapter 4)
            │    ├─ Login Flow
            │    ├─ Settings UI (Chapter 5)
            │    └─ In-Game HUD
            ├─ Character & Input (Chapter 6)
            │    ├─ Character / Pawn
            │    ├─ Input (Enhanced Input)
            │    └─ Camera
            ├─ GAS (Chapter 7)
            │    ├─ ASC / GA / AttributeSet
            │    ├─ Health / Inventory / Interaction
            │    └─ Equipment & Weapons (Chapter 8)
            │         ├─ Cosmetics System (Chapter 9)
            │         ├─ Teams System (Chapter 9)
            │         └─ Indicators (Chapter 9)
            └─ Game Flow (Chapter 10)
                 ├─ Phase / Bot / Accolade
                 ├─ Messages / Cheats / ReplicationGraph
                 ├─ Automated Tests / Replays
                 └─ Bomberman Gameplay
```
