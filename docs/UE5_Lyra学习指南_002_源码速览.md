# UE5_Lyra学习指南_002_源码速览

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)

文章内容由小刚撰写,采用了以下多种方式:

1.口述转文字
2.AI重构
3.参考引擎源码
4.Lyra工程源码
5.结合社区论坛各位大佬的解析

可能存在内容不是完全准确,请以视频内容呈现为最终效果.

- [UE5\_Lyra学习指南\_002\_源码速览](#ue5_lyra学习指南_002_源码速览)
  - [项目目录结构概述](#项目目录结构概述)
    - [LyraGame核心模块](#lyragame核心模块)
      - [能力系统(AbilitySystem)](#能力系统abilitysystem)
      - [角色系统](#角色系统)
      - [游戏功能模块](#游戏功能模块)
      - [交互与反馈](#交互与反馈)
      - [其他关键系统](#其他关键系统)
    - [LyraEditor模块](#lyraeditor模块)
  - [插件架构解析](#插件架构解析)
    - [核心游戏插件](#核心游戏插件)
    - [辅助功能插件](#辅助功能插件)
  - [关键设计理念](#关键设计理念)
  - [学习建议](#学习建议)
  - [项目目录代码结构](#项目目录代码结构)
  - [插件目录代码结构](#插件目录代码结构)

## 项目目录结构概述
Lyra项目采用模块化设计，主要分为两大模块：**游戏模块(LyraGame)**和**编辑器模块(LyraEditor)**。这种结构清晰地分离了运行时逻辑和编辑器专用功能，是大型Unreal项目的标准做法。

### LyraGame核心模块

#### 能力系统(AbilitySystem)
- **Abilities/**: 包含各种游戏能力实现，如跳跃、死亡、重置等核心能力
- **Attributes/**: 定义角色属性集，包括战斗、生命值等属性
- **Executions/**: 伤害和治疗计算逻辑
- **Phases/**: 游戏阶段管理子系统

#### 角色系统
- **Character/**: 基础角色类、移动组件、英雄组件
- **Cosmetics/**: 角色外观定制系统
- **Equipment/**: 装备管理系统和快速栏组件

#### 游戏功能模块
- **GameFeatures/**: 基于GameFeature插件系统的功能扩展
- **GameModes/**: 游戏模式、体验定义和管理系统
- **Inventory/**: 物品库存管理系统

#### 交互与反馈
- **Interaction/**: 游戏内交互系统
- **Feedback/**: 视觉和听觉反馈系统，包括伤害数字弹出效果
- **Messages/**: 游戏消息传递系统

#### 其他关键系统
- **Player/**: 玩家控制器、玩家状态等
- **UI/**: 用户界面系统，包括HUD、加载屏幕等
- **Weapons/**: 武器系统实现

### LyraEditor模块
- **Commandlets/**: 内容验证命令
- **Utilities/**: 编辑器实用工具
- **Validation/**: 蓝图和资源验证系统

## 插件架构解析

Lyra项目通过插件系统实现了高度模块化和可扩展性，主要插件包括：

### 核心游戏插件
1. **CommonGame**: 提供游戏基础框架
   - 通用游戏实例、本地玩家和控制器
   - UI管理系统和布局组件
   - 消息子系统

2. **CommonUser**: 用户管理系统
   - 用户初始化和会话管理
   - 用户状态维护

3. **GameFeatures**: 游戏功能扩展系统
   - **ShooterCore**: 射击游戏核心功能
   - **ShooterExplorer**: AI和交互系统
   - **TopDownArena**: 俯视角竞技场模式

### 辅助功能插件
- **CommonLoadingScreen**: 加载屏幕管理
- **GameSettings**: 游戏设置系统
- **GameplayMessageRouter**: 游戏消息路由系统
- **ModularGameplayActors**: 模块化游戏角色系统
- **UIExtension**: UI扩展系统

## 关键设计理念

1. **模块化分离**：将不同功能明确划分到独立模块中，如将武器系统与库存系统分离，既保持独立性又允许交互。

2. **GameFeature架构**：通过GameFeature插件实现玩法动态加载，支持不同游戏模式的无缝切换。

3. **消息驱动设计**：使用GameplayMessage系统实现松耦合通信，减少直接引用依赖。

4. **扩展性考虑**：通过ModularGameplayActors等系统支持功能扩展而不修改核心类。

## 学习建议

1. 从核心系统开始：先理解AbilitySystem、GameMode等核心模块
2. 关注模块间交互：如Inventory与Equipment系统的协作方式
3. 实践配置流程：项目Config文件夹中的配置对项目运行至关重要
4. 利用生成工具：可使用Python脚本生成项目结构图辅助理解

这种架构设计体现了Epic对于大型游戏项目的最佳实践，值得深入研究和学习其设计思路。
## 项目目录代码结构
```
项目目录代码结构: Source/
├── LyraEditor/
│   ├── Commandlets/
│   │   ├── ContentValidationCommandlet.cpp
│   │   └── ContentValidationCommandlet.h
│   ├── Private/
│   │   ├── AssetTypeActions_LyraContextEffectsLibrary.cpp
│   │   ├── AssetTypeActions_LyraContextEffectsLibrary.h
│   │   ├── GameEditorStyle.cpp
│   │   ├── GameEditorStyle.h
│   │   ├── LyraContextEffectsLibraryFactory.cpp
│   │   └── LyraContextEffectsLibraryFactory.h
│   ├── Utilities/
│   │   ├── CheckChaosMeshCollision.cpp
│   │   ├── CreateRedirectorPackage.cpp
│   │   └── DiffCollectionReferenceSupport.cpp
│   ├── Validation/
│   │   ├── EditorValidator.cpp
│   │   ├── EditorValidator.h
│   │   ├── EditorValidator_Blueprints.cpp
│   │   ├── EditorValidator_Blueprints.h
│   │   ├── EditorValidator_Load.cpp
│   │   ├── EditorValidator_Load.h
│   │   ├── EditorValidator_MaterialFunctions.cpp
│   │   ├── EditorValidator_MaterialFunctions.h
│   │   ├── EditorValidator_SourceControl.cpp
│   │   └── EditorValidator_SourceControl.h
│   ├── LyraEditor.cpp
│   ├── LyraEditor.h
│   ├── LyraEditorEngine.cpp
│   └── LyraEditorEngine.h
├── LyraGame/
│   ├── AbilitySystem/
│   │   ├── Abilities/
│   │   │   ├── LyraAbilityCost.h
│   │   │   ├── LyraAbilityCost_InventoryItem.cpp
│   │   │   ├── LyraAbilityCost_InventoryItem.h
│   │   │   ├── LyraAbilityCost_ItemTagStack.cpp
│   │   │   ├── LyraAbilityCost_ItemTagStack.h
│   │   │   ├── LyraAbilityCost_PlayerTagStack.cpp
│   │   │   ├── LyraAbilityCost_PlayerTagStack.h
│   │   │   ├── LyraAbilitySimpleFailureMessage.h
│   │   │   ├── LyraGameplayAbility.cpp
│   │   │   ├── LyraGameplayAbility.h
│   │   │   ├── LyraGameplayAbility_Death.cpp
│   │   │   ├── LyraGameplayAbility_Death.h
│   │   │   ├── LyraGameplayAbility_Jump.cpp
│   │   │   ├── LyraGameplayAbility_Jump.h
│   │   │   ├── LyraGameplayAbility_Reset.cpp
│   │   │   └── LyraGameplayAbility_Reset.h
│   │   ├── Attributes/
│   │   │   ├── LyraAttributeSet.cpp
│   │   │   ├── LyraAttributeSet.h
│   │   │   ├── LyraCombatSet.cpp
│   │   │   ├── LyraCombatSet.h
│   │   │   ├── LyraHealthSet.cpp
│   │   │   └── LyraHealthSet.h
│   │   ├── Executions/
│   │   │   ├── LyraDamageExecution.cpp
│   │   │   ├── LyraDamageExecution.h
│   │   │   ├── LyraHealExecution.cpp
│   │   │   └── LyraHealExecution.h
│   │   ├── Phases/
│   │   │   ├── LyraGamePhaseAbility.cpp
│   │   │   ├── LyraGamePhaseAbility.h
│   │   │   ├── LyraGamePhaseLog.h
│   │   │   ├── LyraGamePhaseSubsystem.cpp
│   │   │   └── LyraGamePhaseSubsystem.h
│   │   ├── LyraAbilitySet.cpp
│   │   ├── LyraAbilitySet.h
│   │   ├── LyraAbilitySourceInterface.cpp
│   │   ├── LyraAbilitySourceInterface.h
│   │   ├── LyraAbilitySystemComponent.cpp
│   │   ├── LyraAbilitySystemComponent.h
│   │   ├── LyraAbilitySystemGlobals.cpp
│   │   ├── LyraAbilitySystemGlobals.h
│   │   ├── LyraAbilityTagRelationshipMapping.cpp
│   │   ├── LyraAbilityTagRelationshipMapping.h
│   │   ├── LyraGameplayAbilityTargetData_SingleTargetHit.cpp
│   │   ├── LyraGameplayAbilityTargetData_SingleTargetHit.h
│   │   ├── LyraGameplayCueManager.cpp
│   │   ├── LyraGameplayCueManager.h
│   │   ├── LyraGameplayEffectContext.cpp
│   │   ├── LyraGameplayEffectContext.h
│   │   ├── LyraGlobalAbilitySystem.cpp
│   │   ├── LyraGlobalAbilitySystem.h
│   │   ├── LyraTaggedActor.cpp
│   │   └── LyraTaggedActor.h
│   ├── Animation/
│   │   ├── LyraAnimInstance.cpp
│   │   └── LyraAnimInstance.h
│   ├── Audio/
│   │   ├── LyraAudioMixEffectsSubsystem.cpp
│   │   ├── LyraAudioMixEffectsSubsystem.h
│   │   ├── LyraAudioSettings.cpp
│   │   └── LyraAudioSettings.h
│   ├── Camera/
│   │   ├── LyraCameraAssistInterface.h
│   │   ├── LyraCameraComponent.cpp
│   │   ├── LyraCameraComponent.h
│   │   ├── LyraCameraMode.cpp
│   │   ├── LyraCameraMode.h
│   │   ├── LyraCameraMode_ThirdPerson.cpp
│   │   ├── LyraCameraMode_ThirdPerson.h
│   │   ├── LyraPenetrationAvoidanceFeeler.h
│   │   ├── LyraPlayerCameraManager.cpp
│   │   ├── LyraPlayerCameraManager.h
│   │   ├── LyraUICameraManagerComponent.cpp
│   │   └── LyraUICameraManagerComponent.h
│   ├── Character/
│   │   ├── LyraCharacter.cpp
│   │   ├── LyraCharacter.h
│   │   ├── LyraCharacterMovementComponent.cpp
│   │   ├── LyraCharacterMovementComponent.h
│   │   ├── LyraCharacterWithAbilities.cpp
│   │   ├── LyraCharacterWithAbilities.h
│   │   ├── LyraHealthComponent.cpp
│   │   ├── LyraHealthComponent.h
│   │   ├── LyraHeroComponent.cpp
│   │   ├── LyraHeroComponent.h
│   │   ├── LyraPawn.cpp
│   │   ├── LyraPawn.h
│   │   ├── LyraPawnData.cpp
│   │   ├── LyraPawnData.h
│   │   ├── LyraPawnExtensionComponent.cpp
│   │   └── LyraPawnExtensionComponent.h
│   ├── Cosmetics/
│   │   ├── LyraCharacterPartTypes.h
│   │   ├── LyraControllerComponent_CharacterParts.cpp
│   │   ├── LyraControllerComponent_CharacterParts.h
│   │   ├── LyraCosmeticAnimationTypes.cpp
│   │   ├── LyraCosmeticAnimationTypes.h
│   │   ├── LyraCosmeticCheats.cpp
│   │   ├── LyraCosmeticCheats.h
│   │   ├── LyraCosmeticDeveloperSettings.cpp
│   │   ├── LyraCosmeticDeveloperSettings.h
│   │   ├── LyraPawnComponent_CharacterParts.cpp
│   │   └── LyraPawnComponent_CharacterParts.h
│   ├── Development/
│   │   ├── LyraBotCheats.cpp
│   │   ├── LyraBotCheats.h
│   │   ├── LyraDeveloperSettings.cpp
│   │   ├── LyraDeveloperSettings.h
│   │   ├── LyraPlatformEmulationSettings.cpp
│   │   └── LyraPlatformEmulationSettings.h
│   ├── Equipment/
│   │   ├── LyraEquipmentDefinition.cpp
│   │   ├── LyraEquipmentDefinition.h
│   │   ├── LyraEquipmentInstance.cpp
│   │   ├── LyraEquipmentInstance.h
│   │   ├── LyraEquipmentManagerComponent.cpp
│   │   ├── LyraEquipmentManagerComponent.h
│   │   ├── LyraGameplayAbility_FromEquipment.cpp
│   │   ├── LyraGameplayAbility_FromEquipment.h
│   │   ├── LyraPickupDefinition.cpp
│   │   ├── LyraPickupDefinition.h
│   │   ├── LyraQuickBarComponent.cpp
│   │   └── LyraQuickBarComponent.h
│   ├── Feedback/
│   │   ├── ContextEffects/
│   │   │   ├── AnimNotify_LyraContextEffects.cpp
│   │   │   ├── AnimNotify_LyraContextEffects.h
│   │   │   ├── LyraContextEffectComponent.cpp
│   │   │   ├── LyraContextEffectComponent.h
│   │   │   ├── LyraContextEffectsInterface.cpp
│   │   │   ├── LyraContextEffectsInterface.h
│   │   │   ├── LyraContextEffectsLibrary.cpp
│   │   │   ├── LyraContextEffectsLibrary.h
│   │   │   ├── LyraContextEffectsSubsystem.cpp
│   │   │   └── LyraContextEffectsSubsystem.h
│   │   └── NumberPops/
│   │       ├── LyraDamagePopStyle.cpp
│   │       ├── LyraDamagePopStyle.h
│   │       ├── LyraDamagePopStyleNiagara.h
│   │       ├── LyraNumberPopComponent.cpp
│   │       ├── LyraNumberPopComponent.h
│   │       ├── LyraNumberPopComponent_MeshText.cpp
│   │       ├── LyraNumberPopComponent_MeshText.h
│   │       ├── LyraNumberPopComponent_NiagaraText.cpp
│   │       └── LyraNumberPopComponent_NiagaraText.h
│   ├── GameFeatures/
│   │   ├── GameFeatureAction_AddAbilities.cpp
│   │   ├── GameFeatureAction_AddAbilities.h
│   │   ├── GameFeatureAction_AddGameplayCuePath.cpp
│   │   ├── GameFeatureAction_AddGameplayCuePath.h
│   │   ├── GameFeatureAction_AddInputBinding.cpp
│   │   ├── GameFeatureAction_AddInputBinding.h
│   │   ├── GameFeatureAction_AddInputContextMapping.cpp
│   │   ├── GameFeatureAction_AddInputContextMapping.h
│   │   ├── GameFeatureAction_AddWidget.cpp
│   │   ├── GameFeatureAction_AddWidget.h
│   │   ├── GameFeatureAction_SplitscreenConfig.cpp
│   │   ├── GameFeatureAction_SplitscreenConfig.h
│   │   ├── GameFeatureAction_WorldActionBase.cpp
│   │   ├── GameFeatureAction_WorldActionBase.h
│   │   ├── LyraGameFeaturePolicy.cpp
│   │   └── LyraGameFeaturePolicy.h
│   ├── GameModes/
│   │   ├── AsyncAction_ExperienceReady.cpp
│   │   ├── AsyncAction_ExperienceReady.h
│   │   ├── LyraBotCreationComponent.cpp
│   │   ├── LyraBotCreationComponent.h
│   │   ├── LyraExperienceActionSet.cpp
│   │   ├── LyraExperienceActionSet.h
│   │   ├── LyraExperienceDefinition.cpp
│   │   ├── LyraExperienceDefinition.h
│   │   ├── LyraExperienceManager.cpp
│   │   ├── LyraExperienceManager.h
│   │   ├── LyraExperienceManagerComponent.cpp
│   │   ├── LyraExperienceManagerComponent.h
│   │   ├── LyraGameMode.cpp
│   │   ├── LyraGameMode.h
│   │   ├── LyraGameState.cpp
│   │   ├── LyraGameState.h
│   │   ├── LyraUserFacingExperienceDefinition.cpp
│   │   ├── LyraUserFacingExperienceDefinition.h
│   │   ├── LyraWorldSettings.cpp
│   │   └── LyraWorldSettings.h
│   ├── Hotfix/
│   │   ├── LyraHotfixManager.cpp
│   │   ├── LyraHotfixManager.h
│   │   ├── LyraRuntimeOptions.cpp
│   │   ├── LyraRuntimeOptions.h
│   │   ├── LyraTextHotfixConfig.cpp
│   │   └── LyraTextHotfixConfig.h
│   ├── Input/
│   │   ├── LyraAimSensitivityData.cpp
│   │   ├── LyraAimSensitivityData.h
│   │   ├── LyraInputComponent.cpp
│   │   ├── LyraInputComponent.h
│   │   ├── LyraInputConfig.cpp
│   │   ├── LyraInputConfig.h
│   │   ├── LyraInputModifiers.cpp
│   │   ├── LyraInputModifiers.h
│   │   ├── LyraInputUserSettings.cpp
│   │   ├── LyraInputUserSettings.h
│   │   ├── LyraPlayerMappableKeyProfile.cpp
│   │   └── LyraPlayerMappableKeyProfile.h
│   ├── Interaction/
│   │   ├── Abilities/
│   │   │   ├── GameplayAbilityTargetActor_Interact.cpp
│   │   │   ├── GameplayAbilityTargetActor_Interact.h
│   │   │   ├── LyraGameplayAbility_Interact.cpp
│   │   │   └── LyraGameplayAbility_Interact.h
│   │   ├── Tasks/
│   │   │   ├── AbilityTask_GrantNearbyInteraction.cpp
│   │   │   ├── AbilityTask_GrantNearbyInteraction.h
│   │   │   ├── AbilityTask_WaitForInteractableTargets.cpp
│   │   │   ├── AbilityTask_WaitForInteractableTargets.h
│   │   │   ├── AbilityTask_WaitForInteractableTargets_SingleLineTrace.cpp
│   │   │   └── AbilityTask_WaitForInteractableTargets_SingleLineTrace.h
│   │   ├── IInteractableTarget.h
│   │   ├── IInteractionInstigator.h
│   │   ├── InteractionOption.h
│   │   ├── InteractionQuery.h
│   │   ├── InteractionStatics.cpp
│   │   ├── InteractionStatics.h
│   │   └── LyraInteractionDurationMessage.h
│   ├── Inventory/
│   │   ├── IPickupable.cpp
│   │   ├── IPickupable.h
│   │   ├── InventoryFragment_EquippableItem.cpp
│   │   ├── InventoryFragment_EquippableItem.h
│   │   ├── InventoryFragment_PickupIcon.cpp
│   │   ├── InventoryFragment_PickupIcon.h
│   │   ├── InventoryFragment_QuickBarIcon.cpp
│   │   ├── InventoryFragment_QuickBarIcon.h
│   │   ├── InventoryFragment_SetStats.cpp
│   │   ├── InventoryFragment_SetStats.h
│   │   ├── LyraInventoryItemDefinition.cpp
│   │   ├── LyraInventoryItemDefinition.h
│   │   ├── LyraInventoryItemInstance.cpp
│   │   ├── LyraInventoryItemInstance.h
│   │   ├── LyraInventoryManagerComponent.cpp
│   │   └── LyraInventoryManagerComponent.h
│   ├── Messages/
│   │   ├── GameplayMessageProcessor.cpp
│   │   ├── GameplayMessageProcessor.h
│   │   ├── LyraNotificationMessage.cpp
│   │   ├── LyraNotificationMessage.h
│   │   ├── LyraVerbMessage.h
│   │   ├── LyraVerbMessageHelpers.cpp
│   │   ├── LyraVerbMessageHelpers.h
│   │   ├── LyraVerbMessageReplication.cpp
│   │   └── LyraVerbMessageReplication.h
│   ├── Performance/
│   │   ├── LyraMemoryDebugCommands.cpp
│   │   ├── LyraPerformanceSettings.cpp
│   │   ├── LyraPerformanceSettings.h
│   │   ├── LyraPerformanceStatSubsystem.cpp
│   │   ├── LyraPerformanceStatSubsystem.h
│   │   └── LyraPerformanceStatTypes.h
│   ├── Physics/
│   │   ├── LyraCollisionChannels.h
│   │   ├── PhysicalMaterialWithTags.cpp
│   │   └── PhysicalMaterialWithTags.h
│   ├── Player/
│   │   ├── LyraCheatManager.cpp
│   │   ├── LyraCheatManager.h
│   │   ├── LyraDebugCameraController.cpp
│   │   ├── LyraDebugCameraController.h
│   │   ├── LyraLocalPlayer.cpp
│   │   ├── LyraLocalPlayer.h
│   │   ├── LyraPlayerBotController.cpp
│   │   ├── LyraPlayerBotController.h
│   │   ├── LyraPlayerController.cpp
│   │   ├── LyraPlayerController.h
│   │   ├── LyraPlayerSpawningManagerComponent.cpp
│   │   ├── LyraPlayerSpawningManagerComponent.h
│   │   ├── LyraPlayerStart.cpp
│   │   ├── LyraPlayerStart.h
│   │   ├── LyraPlayerState.cpp
│   │   └── LyraPlayerState.h
│   ├── Replays/
│   │   ├── AsyncAction_QueryReplays.cpp
│   │   ├── AsyncAction_QueryReplays.h
│   │   ├── LyraReplaySubsystem.cpp
│   │   └── LyraReplaySubsystem.h
│   ├── Settings/
│   │   ├── CustomSettings/
│   │   │   ├── LyraSettingAction_SafeZoneEditor.cpp
│   │   │   ├── LyraSettingAction_SafeZoneEditor.h
│   │   │   ├── LyraSettingKeyboardInput.cpp
│   │   │   ├── LyraSettingKeyboardInput.h
│   │   │   ├── LyraSettingValueDiscreteDynamic_AudioOutputDevice.cpp
│   │   │   ├── LyraSettingValueDiscreteDynamic_AudioOutputDevice.h
│   │   │   ├── LyraSettingValueDiscrete_Language.cpp
│   │   │   ├── LyraSettingValueDiscrete_Language.h
│   │   │   ├── LyraSettingValueDiscrete_MobileFPSType.cpp
│   │   │   ├── LyraSettingValueDiscrete_MobileFPSType.h
│   │   │   ├── LyraSettingValueDiscrete_OverallQuality.cpp
│   │   │   ├── LyraSettingValueDiscrete_OverallQuality.h
│   │   │   ├── LyraSettingValueDiscrete_PerfStat.cpp
│   │   │   ├── LyraSettingValueDiscrete_PerfStat.h
│   │   │   ├── LyraSettingValueDiscrete_Resolution.cpp
│   │   │   └── LyraSettingValueDiscrete_Resolution.h
│   │   ├── Screens/
│   │   │   ├── LyraBrightnessEditor.cpp
│   │   │   ├── LyraBrightnessEditor.h
│   │   │   ├── LyraSafeZoneEditor.cpp
│   │   │   └── LyraSafeZoneEditor.h
│   │   ├── Widgets/
│   │   │   ├── LyraSettingsListEntrySetting_KeyboardInput.cpp
│   │   │   └── LyraSettingsListEntrySetting_KeyboardInput.h
│   │   ├── LyraGameSettingRegistry.cpp
│   │   ├── LyraGameSettingRegistry.h
│   │   ├── LyraGameSettingRegistry_Audio.cpp
│   │   ├── LyraGameSettingRegistry_Gamepad.cpp
│   │   ├── LyraGameSettingRegistry_Gameplay.cpp
│   │   ├── LyraGameSettingRegistry_MouseAndKeyboard.cpp
│   │   ├── LyraGameSettingRegistry_PerfStats.cpp
│   │   ├── LyraGameSettingRegistry_Video.cpp
│   │   ├── LyraSettingsLocal.cpp
│   │   ├── LyraSettingsLocal.h
│   │   ├── LyraSettingsShared.cpp
│   │   ├── LyraSettingsShared.h
│   ├── System/
│   │   ├── GameplayTagStack.cpp
│   │   ├── GameplayTagStack.h
│   │   ├── LyraActorUtilities.cpp
│   │   ├── LyraActorUtilities.h
│   │   ├── LyraAssetManager.cpp
│   │   ├── LyraAssetManager.h
│   │   ├── LyraAssetManagerStartupJob.cpp
│   │   ├── LyraAssetManagerStartupJob.h
│   │   ├── LyraDevelopmentStatics.cpp
│   │   ├── LyraDevelopmentStatics.h
│   │   ├── LyraGameData.cpp
│   │   ├── LyraGameData.h
│   │   ├── LyraGameEngine.cpp
│   │   ├── LyraGameEngine.h
│   │   ├── LyraGameInstance.cpp
│   │   ├── LyraGameInstance.h
│   │   ├── LyraGameSession.cpp
│   │   ├── LyraGameSession.h
│   │   ├── LyraReplicationGraph.cpp
│   │   ├── LyraReplicationGraph.h
│   │   ├── LyraReplicationGraphSettings.cpp
│   │   ├── LyraReplicationGraphSettings.h
│   │   ├── LyraReplicationGraphTypes.h
│   │   ├── LyraSignificanceManager.cpp
│   │   ├── LyraSignificanceManager.h
│   │   ├── LyraSystemStatics.cpp
│   │   └── LyraSystemStatics.h
│   ├── Teams/
│   │   ├── AsyncAction_ObserveTeam.cpp
│   │   ├── AsyncAction_ObserveTeam.h
│   │   ├── AsyncAction_ObserveTeamColors.cpp
│   │   ├── AsyncAction_ObserveTeamColors.h
│   │   ├── LyraTeamAgentInterface.cpp
│   │   ├── LyraTeamAgentInterface.h
│   │   ├── LyraTeamCheats.cpp
│   │   ├── LyraTeamCheats.h
│   │   ├── LyraTeamCreationComponent.cpp
│   │   ├── LyraTeamCreationComponent.h
│   │   ├── LyraTeamDisplayAsset.cpp
│   │   ├── LyraTeamDisplayAsset.h
│   │   ├── LyraTeamInfoBase.cpp
│   │   ├── LyraTeamInfoBase.h
│   │   ├── LyraTeamPrivateInfo.cpp
│   │   ├── LyraTeamPrivateInfo.h
│   │   ├── LyraTeamPublicInfo.cpp
│   │   ├── LyraTeamPublicInfo.h
│   │   ├── LyraTeamStatics.cpp
│   │   ├── LyraTeamStatics.h
│   │   ├── LyraTeamSubsystem.cpp
│   │   └── LyraTeamSubsystem.h
│   ├── Tests/
│   │   ├── LyraGameplayRpcRegistrationComponent.cpp
│   │   ├── LyraGameplayRpcRegistrationComponent.h
│   │   ├── LyraTestControllerBootTest.cpp
│   │   └── LyraTestControllerBootTest.h
│   ├── UI/
│   │   ├── Basic/
│   │   │   ├── MaterialProgressBar.cpp
│   │   │   └── MaterialProgressBar.h
│   │   ├── Common/
│   │   │   ├── LyraBoundActionButton.cpp
│   │   │   ├── LyraBoundActionButton.h
│   │   │   ├── LyraListView.cpp
│   │   │   ├── LyraListView.h
│   │   │   ├── LyraTabButtonBase.cpp
│   │   │   ├── LyraTabButtonBase.h
│   │   │   ├── LyraTabListWidgetBase.cpp
│   │   │   ├── LyraTabListWidgetBase.h
│   │   │   ├── LyraWidgetFactory.cpp
│   │   │   ├── LyraWidgetFactory.h
│   │   │   ├── LyraWidgetFactory_Class.cpp
│   │   │   └── LyraWidgetFactory_Class.h
│   │   ├── Foundation/
│   │   │   ├── LyraActionWidget.cpp
│   │   │   ├── LyraActionWidget.h
│   │   │   ├── LyraButtonBase.cpp
│   │   │   ├── LyraButtonBase.h
│   │   │   ├── LyraConfirmationScreen.cpp
│   │   │   ├── LyraConfirmationScreen.h
│   │   │   ├── LyraControllerDisconnectedScreen.cpp
│   │   │   ├── LyraControllerDisconnectedScreen.h
│   │   │   ├── LyraLoadingScreenSubsystem.cpp
│   │   │   └── LyraLoadingScreenSubsystem.h
│   │   ├── Frontend/
│   │   │   ├── ApplyFrontendPerfSettingsAction.cpp
│   │   │   ├── ApplyFrontendPerfSettingsAction.h
│   │   │   ├── LyraFrontendStateComponent.cpp
│   │   │   ├── LyraFrontendStateComponent.h
│   │   │   ├── LyraLobbyBackground.cpp
│   │   │   └── LyraLobbyBackground.h
│   │   ├── IndicatorSystem/
│   │   │   ├── IActorIndicatorWidget.h
│   │   │   ├── IndicatorDescriptor.cpp
│   │   │   ├── IndicatorDescriptor.h
│   │   │   ├── IndicatorLayer.cpp
│   │   │   ├── IndicatorLayer.h
│   │   │   ├── IndicatorLibrary.cpp
│   │   │   ├── IndicatorLibrary.h
│   │   │   ├── LyraIndicatorManagerComponent.cpp
│   │   │   ├── LyraIndicatorManagerComponent.h
│   │   │   ├── SActorCanvas.cpp
│   │   │   └── SActorCanvas.h
│   │   ├── PerformanceStats/
│   │   │   ├── LyraPerfStatContainerBase.cpp
│   │   │   ├── LyraPerfStatContainerBase.h
│   │   │   ├── LyraPerfStatWidgetBase.cpp
│   │   │   └── LyraPerfStatWidgetBase.h
│   │   ├── Subsystem/
│   │   │   ├── LyraUIManagerSubsystem.cpp
│   │   │   ├── LyraUIManagerSubsystem.h
│   │   │   ├── LyraUIMessaging.cpp
│   │   │   └── LyraUIMessaging.h
│   │   ├── Weapons/
│   │   │   ├── CircumferenceMarkerWidget.cpp
│   │   │   ├── CircumferenceMarkerWidget.h
│   │   │   ├── HitMarkerConfirmationWidget.cpp
│   │   │   ├── HitMarkerConfirmationWidget.h
│   │   │   ├── LyraReticleWidgetBase.cpp
│   │   │   ├── LyraReticleWidgetBase.h
│   │   │   ├── LyraWeaponUserInterface.cpp
│   │   │   ├── LyraWeaponUserInterface.h
│   │   │   ├── SCircumferenceMarkerWidget.cpp
│   │   │   ├── SCircumferenceMarkerWidget.h
│   │   │   ├── SHitMarkerConfirmationWidget.cpp
│   │   │   └── SHitMarkerConfirmationWidget.h
│   │   ├── LyraActivatableWidget.cpp
│   │   ├── LyraActivatableWidget.h
│   │   ├── LyraGameViewportClient.cpp
│   │   ├── LyraGameViewportClient.h
│   │   ├── LyraHUD.cpp
│   │   ├── LyraHUD.h
│   │   ├── LyraHUDLayout.cpp
│   │   ├── LyraHUDLayout.h
│   │   ├── LyraJoystickWidget.cpp
│   │   ├── LyraJoystickWidget.h
│   │   ├── LyraSettingScreen.cpp
│   │   ├── LyraSettingScreen.h
│   │   ├── LyraSimulatedInputWidget.cpp
│   │   ├── LyraSimulatedInputWidget.h
│   │   ├── LyraTaggedWidget.cpp
│   │   ├── LyraTaggedWidget.h
│   │   ├── LyraTouchRegion.cpp
│   │   └── LyraTouchRegion.h
│   ├── Weapons/
│   │   ├── InventoryFragment_ReticleConfig.cpp
│   │   ├── InventoryFragment_ReticleConfig.h
│   │   ├── LyraDamageLogDebuggerComponent.cpp
│   │   ├── LyraDamageLogDebuggerComponent.h
│   │   ├── LyraGameplayAbility_RangedWeapon.cpp
│   │   ├── LyraGameplayAbility_RangedWeapon.h
│   │   ├── LyraRangedWeaponInstance.cpp
│   │   ├── LyraRangedWeaponInstance.h
│   │   ├── LyraWeaponDebugSettings.cpp
│   │   ├── LyraWeaponDebugSettings.h
│   │   ├── LyraWeaponInstance.cpp
│   │   ├── LyraWeaponInstance.h
│   │   ├── LyraWeaponSpawner.cpp
│   │   ├── LyraWeaponSpawner.h
│   │   ├── LyraWeaponStateComponent.cpp
│   │   └── LyraWeaponStateComponent.h
│   ├── LyraGameModule.cpp
│   ├── LyraGameplayTags.cpp
│   ├── LyraGameplayTags.h
│   ├── LyraLogChannels.cpp
│   ├── LyraLogChannels.h
```
## 插件目录代码结构
```
插件目录代码结构: Plugins/
├── AsyncMixin/
│   ├── Source/
│   │   ├── Private/
│   │   │   ├── AsyncMixin.cpp
│   │   │   └── AsyncMixinModule.cpp
│   │   ├── Public/
│   │   │   └── AsyncMixin.h
├── CommonGame/
│   ├── Source/
│   │   ├── Private/
│   │   │   ├── Actions/
│   │   │   │   ├── AsyncAction_CreateWidgetAsync.cpp
│   │   │   │   ├── AsyncAction_PushContentToLayerForPlayer.cpp
│   │   │   │   └── AsyncAction_ShowConfirmation.cpp
│   │   │   ├── Messaging/
│   │   │   │   ├── CommonGameDialog.cpp
│   │   │   │   └── CommonMessagingSubsystem.cpp
│   │   │   ├── CommonGameInstance.cpp
│   │   │   ├── CommonGameModule.cpp
│   │   │   ├── CommonLocalPlayer.cpp
│   │   │   ├── CommonPlayerController.cpp
│   │   │   ├── CommonPlayerInputKey.cpp
│   │   │   ├── CommonUIExtensions.cpp
│   │   │   ├── GameUIManagerSubsystem.cpp
│   │   │   ├── GameUIPolicy.cpp
│   │   │   ├── LogCommonGame.cpp
│   │   │   ├── LogCommonGame.h
│   │   │   └── PrimaryGameLayout.cpp
│   │   ├── Public/
│   │   │   ├── Actions/
│   │   │   │   ├── AsyncAction_CreateWidgetAsync.h
│   │   │   │   ├── AsyncAction_PushContentToLayerForPlayer.h
│   │   │   │   └── AsyncAction_ShowConfirmation.h
│   │   │   ├── Messaging/
│   │   │   │   ├── CommonGameDialog.h
│   │   │   │   └── CommonMessagingSubsystem.h
│   │   │   ├── CommonGameInstance.h
│   │   │   ├── CommonLocalPlayer.h
│   │   │   ├── CommonPlayerController.h
│   │   │   ├── CommonPlayerInputKey.h
│   │   │   ├── CommonUIExtensions.h
│   │   │   ├── GameUIManagerSubsystem.h
│   │   │   ├── GameUIPolicy.h
│   │   │   └── PrimaryGameLayout.h
├── CommonLoadingScreen/
│   ├── Resources/
│   ├── Source/
│   │   ├── CommonLoadingScreen/
│   │   │   ├── Private/
│   │   │   │   ├── CommonLoadingScreenModule.cpp
│   │   │   │   ├── CommonLoadingScreenSettings.cpp
│   │   │   │   ├── CommonLoadingScreenSettings.h
│   │   │   │   └── LoadingScreenManager.cpp
│   │   │   ├── Public/
│   │   │   │   ├── LoadingProcessInterface.h
│   │   │   │   ├── LoadingProcessTask.cpp
│   │   │   │   ├── LoadingProcessTask.h
│   │   │   │   └── LoadingScreenManager.h
│   │   └── CommonStartupLoadingScreen/
│   │       ├── Private/
│   │       │   ├── CommonPreLoadScreen.cpp
│   │       │   ├── CommonPreLoadScreen.h
│   │       │   ├── CommonStartupLoadingScreen.cpp
│   │       │   ├── SCommonPreLoadingScreenWidget.cpp
│   │       │   └── SCommonPreLoadingScreenWidget.h
├── CommonUser/
│   ├── Resources/
│   ├── Source/
│   │   └── CommonUser/
│   │       ├── Private/
│   │       │   ├── AsyncAction_CommonUserInitialize.cpp
│   │       │   ├── CommonSessionSubsystem.cpp
│   │       │   ├── CommonUserBasicPresence.cpp
│   │       │   ├── CommonUserModule.cpp
│   │       │   ├── CommonUserSubsystem.cpp
│   │       │   └── CommonUserTypes.cpp
│   │       ├── Public/
│   │       │   ├── AsyncAction_CommonUserInitialize.h
│   │       │   ├── CommonSessionSubsystem.h
│   │       │   ├── CommonUserBasicPresence.h
│   │       │   ├── CommonUserModule.h
│   │       │   ├── CommonUserSubsystem.h
│   │       │   └── CommonUserTypes.h
├── GameFeatures/
│   ├── ShooterCore/
│   │   ├── Config/
│   │   │   └── Tags/
│   │   ├── Content/
│   │   │   ├── Accolades/
│   │   │   ├── Blueprint/
│   │   │   │   ├── Macros/
│   │   │   ├── Bot/
│   │   │   │   ├── BT/
│   │   │   │   ├── EQS/
│   │   │   │   │   ├── Context/
│   │   │   │   │   ├── Generator/
│   │   │   │   ├── Services/
│   │   │   ├── Camera/
│   │   │   ├── ControlPoint/
│   │   │   │   ├── AI/
│   │   │   │   ├── UI/
│   │   │   ├── Effects/
│   │   │   │   ├── Environmental/
│   │   │   │   ├── Material/
│   │   │   │   └── Mesh/
│   │   │   ├── Elimination/
│   │   │   │   ├── UI/
│   │   │   ├── Experiences/
│   │   │   │   ├── Phases/
│   │   │   ├── Game/
│   │   │   │   ├── Dash/
│   │   │   │   ├── Emote/
│   │   │   │   ├── Melee/
│   │   │   │   ├── Respawn/
│   │   │   ├── GameplayCues/
│   │   │   ├── Input/
│   │   │   │   ├── Abilities/
│   │   │   │   ├── Actions/
│   │   │   │   ├── Components/
│   │   │   │   ├── Mappings/
│   │   │   │   ├── Settings/
│   │   │   ├── Item/
│   │   │   ├── Items/
│   │   │   │   ├── HealthPickup/
│   │   │   │   ├── HealthPickup_Unused/
│   │   │   │   └── PhysicsCube/
│   │   │   ├── Maps/
│   │   │   ├── System/
│   │   │   │   ├── Audio/
│   │   │   │   └── Playlists/
│   │   │   ├── UserInterface/
│   │   │   │   ├── HUD/
│   │   │   │   ├── Notifications/
│   │   │   │   │   ├── Accolades/
│   │   │   │   │   └── EliminationFeed/
│   │   │   │   ├── Replays/
│   │   │   │   ├── ScoreBoard/
│   │   │   │   │   ├── PlayerRow/
│   │   │   ├── Weapon/
│   │   │   │   └── Grenade/
│   │   │   ├── Weapons/
│   │   │   │   ├── Grenade/
│   │   │   │   ├── NetShooter_PROTO/
│   │   │   │   ├── Pistol/
│   │   │   │   ├── Rifle/
│   │   │   │   ├── Shotgun/
│   │   ├── Resources/
│   │   ├── Source/
│   │   │   └── ShooterCoreRuntime/
│   │   │       ├── Private/
│   │   │       │   ├── Accolades/
│   │   │       │   │   ├── LyraAccoladeDefinition.cpp
│   │   │       │   │   └── LyraAccoladeHostWidget.cpp
│   │   │       │   ├── Input/
│   │   │       │   │   ├── AimAssistInputModifier.cpp
│   │   │       │   │   ├── AimAssistTargetComponent.cpp
│   │   │       │   │   └── AimAssistTargetManagerComponent.cpp
│   │   │       │   ├── MessageProcessors/
│   │   │       │   │   ├── AssistProcessor.cpp
│   │   │       │   │   ├── ElimChainProcessor.cpp
│   │   │       │   │   └── ElimStreakProcessor.cpp
│   │   │       │   ├── LyraWorldCollectable.cpp
│   │   │       │   ├── ShooterCoreRuntimeModule.cpp
│   │   │       │   ├── ShooterCoreRuntimeSettings.cpp
│   │   │       │   ├── TDM_PlayerSpawningManagmentComponent.cpp
│   │   │       │   └── TDM_PlayerSpawningManagmentComponent.h
│   │   │       ├── Public/
│   │   │       │   ├── Accolades/
│   │   │       │   │   ├── LyraAccoladeDefinition.h
│   │   │       │   │   └── LyraAccoladeHostWidget.h
│   │   │       │   ├── Input/
│   │   │       │   │   ├── AimAssistInputModifier.h
│   │   │       │   │   ├── AimAssistTargetComponent.h
│   │   │       │   │   ├── AimAssistTargetManagerComponent.h
│   │   │       │   │   └── IAimAssistTargetInterface.h
│   │   │       │   ├── MessageProcessors/
│   │   │       │   │   ├── AssistProcessor.h
│   │   │       │   │   ├── ElimChainProcessor.h
│   │   │       │   │   └── ElimStreakProcessor.h
│   │   │       │   ├── Messages/
│   │   │       │   │   └── ControlPointStatusMessage.h
│   │   │       │   ├── LyraWorldCollectable.h
│   │   │       │   ├── ShooterCoreRuntimeModule.h
│   │   │       │   └── ShooterCoreRuntimeSettings.h
│   ├── ShooterExplorer/
│   │   ├── Config/
│   │   │   └── Tags/
│   │   ├── Content/
│   │   │   ├── AI/
│   │   │   │   ├── BehaviorTrees/
│   │   │   │   │   ├── BB/
│   │   │   │   │   ├── EnvQueries/
│   │   │   │   │   ├── Tasks/
│   │   │   │   ├── SmartObjects/
│   │   │   │   │   ├── CAS_Data/
│   │   │   │   │   ├── Definition/
│   │   │   │   │   ├── Parents/
│   │   │   │   │   ├── StateTrees/
│   │   │   │   ├── StateTrees/
│   │   │   │   │   ├── Evaluators/
│   │   │   │   │   ├── Tasks/
│   │   │   │   │   └── Trees/
│   │   │   │   └── UnrealFestDemo/
│   │   │   ├── Blueprint/
│   │   │   ├── Game/
│   │   │   ├── Input/
│   │   │   │   ├── Abilities/
│   │   │   │   ├── Actions/
│   │   │   │   └── Mappings/
│   │   │   ├── Interact/
│   │   │   ├── Items/
│   │   │   ├── Maps/
│   │   │   ├── Meshes/
│   │   │   │   ├── Gyms/
│   │   │   │   └── Prop/
│   │   │   │       └── Kit_bench_RR/
│   │   │   │           ├── Material/
│   │   │   │           ├── Texture/
│   │   │   │           │   ├── parkBench/
│   │   │   │           │   ├── pierBench/
│   │   │   │           └── mesh/
│   │   │   ├── System/
│   │   │   │   ├── DataLayers/
│   │   │   │   └── Experiences/
│   │   │   ├── UserInterface/
│   ├── ShooterMaps/
│   │   ├── Content/
│   │   │   ├── GameplayCues/
│   │   │   ├── Items/
│   │   │   │   └── Backgrounds/
│   │   │   ├── LevelSequence/
│   │   │   ├── Maps/
│   │   │   │   ├── DataLayers/
│   │   │   │   │   ├── L_Convolution_Blockout/
│   │   │   │   │   ├── L_Expanse/
│   │   │   │   │   ├── L_Expanse_Blockout/
│   │   │   │   │   └── L_FiringRange_WP/
│   │   │   ├── Meshes/
│   │   │   │   ├── Convolution_Blockout/
│   │   │   │   ├── Expanse/
│   │   │   │   ├── Expanse_Blockout/
│   │   │   │   ├── FiringRange/
│   │   │   ├── System/
│   │   │   │   └── Playlists/
│   │   ├── Resources/
│   ├── ShooterTests/
│   │   ├── Content/
│   │   │   ├── Blueprint/
│   │   │   ├── Input/
│   │   │   │   ├── DeviceColor/
│   │   │   │   ├── SoundVibrations/
│   │   │   │   ├── TriggerFeedback/
│   │   │   │   ├── TriggerVibration/
│   │   │   ├── Maps/
│   │   │   ├── System/
│   │   │   │   └── Experiences/
│   │   ├── Resources/
│   │   ├── Source/
│   │   │   └── ShooterTestsRuntime/
│   │   │       ├── Private/
│   │   │       │   ├── Utilities/
│   │   │       │   │   ├── ShooterTestsActorNetworkTest.h
│   │   │       │   │   ├── ShooterTestsActorTest.h
│   │   │       │   │   ├── ShooterTestsActorTestHelper.cpp
│   │   │       │   │   ├── ShooterTestsActorTestHelper.h
│   │   │       │   │   ├── ShooterTestsAnimationTestHelper.cpp
│   │   │       │   │   ├── ShooterTestsAnimationTestHelper.h
│   │   │       │   │   ├── ShooterTestsInputTestHelper.cpp
│   │   │       │   │   ├── ShooterTestsInputTestHelper.h
│   │   │       │   │   └── ShooterTestsNetworkComponent.h
│   │   │       │   ├── ShooterTestsActorAnimationTests.cpp
│   │   │       │   ├── ShooterTestsActorNetworkTests.cpp
│   │   │       │   ├── ShooterTestsDevicePropertyTester.cpp
│   │   │       │   ├── ShooterTestsDevicePropertyTester.h
│   │   │       │   ├── ShooterTestsMapTests.cpp
│   │   │       │   └── ShooterTestsRuntimeModule.cpp
│   └── TopDownArena/
│       ├── Config/
│       │   └── Tags/
│       ├── Content/
│       │   ├── Game/
│       │   │   ├── Bombs/
│       │   │   ├── Effects/
│       │   │   │   ├── Bomb/
│       │   │   │   ├── PickUp/
│       │   │   │   ├── Player/
│       │   │   ├── Environment/
│       │   │   ├── MapsGenerator/
│       │   │   ├── Modes/
│       │   │   ├── Pickups/
│       │   │   ├── Powerups/
│       │   ├── GameplayCues/
│       │   ├── Maps/
│       │   ├── System/
│       │   │   ├── Experiences/
│       │   │   └── Playlists/
│       │   ├── UserInterface/
│       │   │   ├── Player/
│       │   │   │   ├── Texture/
│       ├── Resources/
│       ├── Source/
│       │   └── TopDownArenaRuntime/
│       │       ├── Private/
│       │       │   ├── LyraCameraMode_TopDownArenaCamera.cpp
│       │       │   ├── TopDownArenaAttributeSet.cpp
│       │       │   ├── TopDownArenaAttributeSet.h
│       │       │   ├── TopDownArenaMovementComponent.cpp
│       │       │   ├── TopDownArenaMovementComponent.h
│       │       │   ├── TopDownArenaPickupUIData.cpp
│       │       │   ├── TopDownArenaPickupUIData.h
│       │       │   └── TopDownArenaRuntimeModule.cpp
│       │       ├── Public/
│       │       │   ├── LyraCameraMode_TopDownArenaCamera.h
│       │       │   └── TopDownArenaRuntimeModule.h
├── GameSettings/
│   ├── Source/
│   │   ├── Private/
│   │   │   ├── DataSource/
│   │   │   │   └── GameSettingDataSourceDynamic.cpp
│   │   │   ├── EditCondition/
│   │   │   │   ├── WhenPlatformHasTrait.cpp
│   │   │   │   └── WhenPlayingAsPrimaryPlayer.cpp
│   │   │   ├── Registry/
│   │   │   │   ├── GameSettingRegistry.cpp
│   │   │   │   └── GameSettingRegistryChangeTracker.cpp
│   │   │   ├── Widgets/
│   │   │   │   ├── Misc/
│   │   │   │   │   ├── GameSettingPressAnyKey.cpp
│   │   │   │   │   ├── GameSettingRotator.cpp
│   │   │   │   │   └── KeyAlreadyBoundWarning.cpp
│   │   │   │   ├── Responsive/
│   │   │   │   │   ├── GameResponsivePanel.cpp
│   │   │   │   │   ├── GameResponsivePanel.h
│   │   │   │   │   ├── GameResponsivePanelSlot.cpp
│   │   │   │   │   ├── GameResponsivePanelSlot.h
│   │   │   │   │   ├── SGameResponsivePanel.cpp
│   │   │   │   │   └── SGameResponsivePanel.h
│   │   │   │   ├── GameSettingDetailExtension.cpp
│   │   │   │   ├── GameSettingDetailView.cpp
│   │   │   │   ├── GameSettingListEntry.cpp
│   │   │   │   ├── GameSettingListView.cpp
│   │   │   │   ├── GameSettingPanel.cpp
│   │   │   │   ├── GameSettingScreen.cpp
│   │   │   │   ├── GameSettingVisualData.cpp
│   │   │   │   └── IGameSettingActionInterface.cpp
│   │   │   ├── GameSetting.cpp
│   │   │   ├── GameSettingAction.cpp
│   │   │   ├── GameSettingCollection.cpp
│   │   │   ├── GameSettingFilterState.cpp
│   │   │   ├── GameSettingValue.cpp
│   │   │   ├── GameSettingValueDiscrete.cpp
│   │   │   ├── GameSettingValueDiscreteDynamic.cpp
│   │   │   ├── GameSettingValueScalar.cpp
│   │   │   ├── GameSettingValueScalarDynamic.cpp
│   │   │   └── GameSettingsModule.cpp
│   │   ├── Public/
│   │   │   ├── DataSource/
│   │   │   │   ├── GameSettingDataSource.h
│   │   │   │   └── GameSettingDataSourceDynamic.h
│   │   │   ├── EditCondition/
│   │   │   │   ├── WhenCondition.h
│   │   │   │   ├── WhenPlatformHasTrait.h
│   │   │   │   └── WhenPlayingAsPrimaryPlayer.h
│   │   │   ├── Widgets/
│   │   │   │   ├── Misc/
│   │   │   │   │   ├── GameSettingPressAnyKey.h
│   │   │   │   │   ├── GameSettingRotator.h
│   │   │   │   │   └── KeyAlreadyBoundWarning.h
│   │   │   │   ├── GameSettingDetailExtension.h
│   │   │   │   ├── GameSettingDetailView.h
│   │   │   │   ├── GameSettingListEntry.h
│   │   │   │   ├── GameSettingListView.h
│   │   │   │   ├── GameSettingPanel.h
│   │   │   │   ├── GameSettingScreen.h
│   │   │   │   ├── GameSettingVisualData.h
│   │   │   │   └── IGameSettingActionInterface.h
│   │   │   ├── GameSetting.h
│   │   │   ├── GameSettingAction.h
│   │   │   ├── GameSettingCollection.h
│   │   │   ├── GameSettingFilterState.h
│   │   │   ├── GameSettingRegistry.h
│   │   │   ├── GameSettingRegistryChangeTracker.h
│   │   │   ├── GameSettingValue.h
│   │   │   ├── GameSettingValueDiscrete.h
│   │   │   ├── GameSettingValueDiscreteDynamic.h
│   │   │   ├── GameSettingValueScalar.h
│   │   │   └── GameSettingValueScalarDynamic.h
├── GameSubtitles/
│   ├── Config/
│   │   └── Tags/
│   ├── Source/
│   │   ├── Private/
│   │   │   ├── Players/
│   │   │   │   └── MediaSubtitlesPlayer.cpp
│   │   │   ├── Widgets/
│   │   │   │   ├── SSubtitleDisplay.cpp
│   │   │   │   └── SubtitleDisplay.cpp
│   │   │   ├── GameSubtitlesModule.cpp
│   │   │   └── SubtitleDisplaySubsystem.cpp
│   │   ├── Public/
│   │   │   ├── Players/
│   │   │   │   └── MediaSubtitlesPlayer.h
│   │   │   ├── Widgets/
│   │   │   │   ├── SSubtitleDisplay.h
│   │   │   │   └── SubtitleDisplay.h
│   │   │   ├── SubtitleDisplayOptions.h
│   │   │   └── SubtitleDisplaySubsystem.h
├── GameplayMessageRouter/
│   ├── Source/
│   │   ├── GameplayMessageNodes/
│   │   │   ├── Private/
│   │   │   │   ├── GameplayMessageNodesModule.cpp
│   │   │   │   └── K2Node_AsyncAction_ListenForGameplayMessages.cpp
│   │   │   ├── Public/
│   │   │   │   └── K2Node_AsyncAction_ListenForGameplayMessages.h
│   │   └── GameplayMessageRuntime/
│   │       ├── Private/
│   │       │   └── GameFramework/
│   │       │       ├── AsyncAction_ListenForGameplayMessage.cpp
│   │       │       ├── GameplayMessageRuntime.cpp
│   │       │       └── GameplayMessageSubsystem.cpp
│   │       ├── Public/
│   │       │   └── GameFramework/
│   │       │       ├── AsyncAction_ListenForGameplayMessage.h
│   │       │       ├── GameplayMessageSubsystem.h
│   │       │       └── GameplayMessageTypes2.h
├── LyraExampleContent/
│   ├── Content/
│   │   ├── MSPresets/
│   │   │   ├── MSTextures/
│   │   │   └── M_MS_Surface_Material/
│   │   │       ├── Functions/
│   │   │       ├── SubSurfaceProfile/
│   │   ├── Materials/
│   │   │   ├── MF/
│   │   │   ├── Textures/
│   │   │   │   ├── GlassPattern/
│   │   └── Megascans/
│   │       └── Surfaces/
│   │           ├── Dirty_Painted_Concrete_Wall_vdqfcg3/
│   │           └── Fortified_Shelter_Wall_ubimbhrdy/
├── LyraExtTool/
│   ├── Content/
│   ├── Resources/
│   ├── Source/
│   │   └── LyraExtTool/
│   │       ├── Private/
│   │       │   ├── BPFunctionLibrary.cpp
│   │       │   └── LyraExtTool.cpp
│   │       ├── Public/
│   │       │   ├── BPFunctionLibrary.h
│   │       │   └── LyraExtTool.h
├── ModularGameplayActors/
│   ├── Source/
│   │   └── ModularGameplayActors/
│   │       ├── Private/
│   │       │   ├── ModularAIController.cpp
│   │       │   ├── ModularCharacter.cpp
│   │       │   ├── ModularGameMode.cpp
│   │       │   ├── ModularGameState.cpp
│   │       │   ├── ModularGameplayActorsModule.cpp
│   │       │   ├── ModularPawn.cpp
│   │       │   ├── ModularPlayerController.cpp
│   │       │   └── ModularPlayerState.cpp
│   │       ├── Public/
│   │       │   ├── ModularAIController.h
│   │       │   ├── ModularCharacter.h
│   │       │   ├── ModularGameMode.h
│   │       │   ├── ModularGameState.h
│   │       │   ├── ModularPawn.h
│   │       │   ├── ModularPlayerController.h
│   │       │   └── ModularPlayerState.h
├── PocketWorlds/
│   ├── Content/
│   ├── Source/
│   │   ├── Private/
│   │   │   ├── PocketCapture.cpp
│   │   │   ├── PocketCaptureSubsystem.cpp
│   │   │   ├── PocketLevel.cpp
│   │   │   ├── PocketLevelInstance.cpp
│   │   │   ├── PocketLevelSystem.cpp
│   │   │   └── PocketWorldsModule.cpp
│   │   ├── Public/
│   │   │   ├── PocketCapture.h
│   │   │   ├── PocketCaptureSubsystem.h
│   │   │   ├── PocketLevel.h
│   │   │   ├── PocketLevelInstance.h
│   │   │   └── PocketLevelSystem.h
└── UIExtension/
    ├── Source/
    │   ├── Private/
    │   │   ├── Widgets/
    │   │   │   └── UIExtensionPointWidget.cpp
    │   │   ├── LogUIExtension.cpp
    │   │   ├── LogUIExtension.h
    │   │   ├── UIExtensionModule.cpp
    │   │   └── UIExtensionSystem.cpp
    │   ├── Public/
    │   │   ├── Widgets/
    │   │   │   └── UIExtensionPointWidget.h
    │   │   └── UIExtensionSystem.h
```