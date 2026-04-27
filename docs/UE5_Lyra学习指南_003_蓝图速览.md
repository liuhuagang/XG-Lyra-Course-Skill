# UE5_Lyra学习指南_003_蓝图速览

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

- [UE5\_Lyra学习指南\_003\_蓝图速览](#ue5_lyra学习指南_003_蓝图速览)
    - [1. 核心框架](#1-核心框架)
    - [2. 角色系统](#2-角色系统)
    - [3. 能力与交互](#3-能力与交互)
    - [4. 资源管理](#4-资源管理)
    - [5. 插件扩展](#5-插件扩展)
    - [6. 工具与测试](#6-工具与测试)
    - [项目目录蓝图结构](#项目目录蓝图结构)
    - [插件目录蓝图结构](#插件目录蓝图结构)

### 1. 核心框架
- **Game Data & Instance**  
  - `DefaultGameData`：硬编码加载的默认资产（Tag、伤害/治疗效果模板）。  
  - `GameInstance`：全局状态管理，继承自C++类。  
- **Game Mode**  
  玩法规则基类，支持多模式扩展（如射击、俯视角）。  
- **打包与配置**  
  `Label`资产用于划分打包模块，`InputConfig`管理增强输入映射。

---

### 2. 角色系统
- **Character**  
  - 基类蓝图：骨骼网格体与动画逻辑分离，通过`CurveMod`驱动动画混合。  
  - `CatMesh`组件：动态换装系统（男/女角色、武器挂载）。  
  - 物理材质标识：`WeakSpot`（头部/手部）触发额外伤害乘数。  
- **Camera**  
  - 堆栈式管理（`LyraCameraMode`），非弹簧臂实现。  
  - 曲线调整相机位置，支持击中反馈等特效。

---

### 3. 能力与交互
- **GAS (Gameplay Ability System)**  
  - 通用能力：跳跃、治疗、装弹（`GA_`前缀蓝图）。  
  - 动画通知：`AnimNotify`触发攻击命中判定。  
- **Enhanced Input**  
  动态键位绑定（如武器开火映射到鼠标/手柄），通过`InputTag`解耦逻辑。

---

### 4. 资源管理
- **Audio**  
  - `MetaSound`：参数化音频系统，动态调节音效属性。  
  - `Submix`：总线控制环境混响与并发限制。  
- **Materials & FX**  
  - 材质函数复用（武器光泽、血迹贴花）。  
  - 粒子特效：击杀数字、受击方向指示（`DamageDirection`曲线）。  

---

### 5. 插件扩展
- **Shooter Core**  
  射击玩法核心（武器逻辑、AI行为树/EQS）。  
- **Experience**  
  动态挂载玩法模块（如团队竞技、生存模式）。  
- **UI System**  
  - `CommonUI`：堆栈式界面管理（主菜单、HUD）。  
  - 本地化支持与设置面板（C++定义功能+蓝图UI）。  

---

### 6. 工具与测试
- **Editor Utilities**  
  开发辅助工具（资产验证、快捷调试设置）。  
- **Test Assets**  
  - 几何体工具：快速构建场景原型。  
  - 回放系统：记录游戏过程。  

---


### 项目目录蓝图结构
```
项目目录蓝图结构: Content/
├── Audio/
│   ├── AttenuationPresets/
│   │   ├── ATT_Amb_Pad.uasset
│   │   ├── ATT_Default.uasset
│   │   ├── ATT_FX_BulletImpact.uasset
│   │   ├── ATT_FX_Dash.uasset
│   │   ├── ATT_FX_GrenadeBounce.uasset
│   │   ├── ATT_FX_Pad.uasset
│   │   ├── ATT_Foley.uasset
│   │   ├── ATT_Footstep_NPC.uasset
│   │   ├── ATT_Footstep_PC.uasset
│   │   ├── ATT_Grenade.uasset
│   │   ├── ATT_Melee.uasset
│   │   ├── ATT_Pistol.uasset
│   │   ├── ATT_Projectile.uasset
│   │   ├── ATT_Rifle.uasset
│   │   ├── ATT_Shotgun.uasset
│   │   ├── ATT_WhizBy.uasset
│   │   └── TempITDSpatializationSourceSettings.uasset
│   ├── Blueprints/
│   │   ├── B_MusicManagerComponent_Base.uasset
│   │   ├── B_MusicManagerComponent_FE.uasset
│   │   ├── B_WindSystem.uasset
│   │   ├── GeneralAudioFunctions.uasset
│   │   └── WeaponAudioMacros.uasset
│   ├── Classes/
│   │   ├── Application_Focused.uasset
│   │   ├── Music.uasset
│   │   ├── Overall.uasset
│   │   ├── RenderedCinematics.uasset
│   │   ├── SFX.uasset
│   │   ├── UI.uasset
│   │   └── VoiceChat.uasset
│   ├── Concurrency/
│   │   ├── SCON_Footsteps.uasset
│   │   ├── SCON_Guns_LimitToOwner.uasset
│   │   ├── SCON_Guns_StopFarthest.uasset
│   │   ├── SCON_Impacts.uasset
│   │   ├── SCON_PlayerDamage.uasset
│   │   ├── SCON_WhizBys.uasset
│   │   └── SCon_Default.uasset
│   ├── Effects/
│   │   └── SubmixEffects/
│   │       ├── CREV_1978_LargeRoom.uasset
│   │       ├── CREV_BaseMain.uasset
│   │       ├── CREV_BaseMain_Tunnel.uasset
│   │       ├── CREV_BaseWing.uasset
│   │       ├── CREV_CenterCylinder.uasset
│   │       ├── CREV_ControlPoint.uasset
│   │       ├── CREV_Exterior.uasset
│   │       ├── DYN_LowDynamics.uasset
│   │       ├── DYN_MainDynamics.uasset
│   │       ├── FLT_EarlyReflections.uasset
│   │       ├── FLT_EarlyReflectionsHPF.uasset
│   │       └── TAP_EarlyReflections.uasset
│   ├── Impulses/
│   │   ├── 1978_LargeRoom.uasset
│   │   ├── 1978_LargeRoom_IR.uasset
│   │   ├── IR_Reverb_Exterior_01.uasset
│   │   ├── IR_Reverb_Exterior_01_IR.uasset
│   │   ├── IR_Reverb_Hall_01_bright.uasset
│   │   ├── IR_Reverb_Hall_01_bright_IR.uasset
│   │   ├── IR_Reverb_Hall_01_dark.uasset
│   │   ├── IR_Reverb_Hall_01_dark_IR.uasset
│   │   ├── IR_Reverb_Hall_02.uasset
│   │   ├── IR_Reverb_Hall_02_IR.uasset
│   │   ├── IR_Reverb_Hall_03.uasset
│   │   ├── IR_Reverb_Hall_03_IR.uasset
│   │   ├── IR_Reverb_Room_01.uasset
│   │   ├── IR_Reverb_Room_01_IR.uasset
│   │   ├── IR_Reverb_Room_02.uasset
│   │   ├── IR_Reverb_Room_02_IR.uasset
│   │   ├── IR_Reverb_Room_03.uasset
│   │   ├── IR_Reverb_Room_03_IR.uasset
│   │   ├── IR_Reverb_Room_04.uasset
│   │   ├── IR_Reverb_Room_04_IR.uasset
│   │   ├── IR_Reverb_Room_05.uasset
│   │   ├── IR_Reverb_Room_05_IR.uasset
│   │   ├── IR_Reverb_Room_06.uasset
│   │   ├── IR_Reverb_Room_06_IR.uasset
│   │   ├── IR_Reverb_Room_07.uasset
│   │   └── IR_Reverb_Room_07_IR.uasset
│   ├── MetaSounds/
│   │   ├── MS_Graph_RandomPitch_Stereo.uasset
│   │   ├── MS_Graph_TriggerDelayPitchShift_Mono.uasset
│   │   ├── lib_DovetailClip.uasset
│   │   ├── lib_DovetailClipFromArray.uasset
│   │   ├── lib_RandInterpTo.uasset
│   │   ├── lib_RandPanStereo.uasset
│   │   ├── lib_StereoBalance.uasset
│   │   ├── lib_TriggerAfter.uasset
│   │   ├── lib_TriggerEvery.uasset
│   │   ├── lib_TriggerModulo.uasset
│   │   ├── lib_TriggerStopAfter.uasset
│   │   ├── lib_WhizBy.uasset
│   │   ├── mx_PlayAmbientChord.uasset
│   │   ├── mx_PlayAmbientElement.uasset
│   │   ├── mx_Stingers.uasset
│   │   ├── mx_System.uasset
│   │   ├── sfx_Amb_Teleport_lp_meta.uasset
│   │   ├── sfx_Amb_WeaponPad_lp_meta.uasset
│   │   ├── sfx_Amb_Wind_lp_meta.uasset
│   │   ├── sfx_BaseLayer_Interactable_Pad_nl_meta.uasset
│   │   ├── sfx_CapturePoint_Progress_meta.uasset
│   │   ├── sfx_Character_DamageGivenKill_nl_meta.uasset
│   │   ├── sfx_Character_DamageGivenWeakSpot_nl_meta.uasset
│   │   ├── sfx_Character_DamageGiven_nl_meta.uasset
│   │   ├── sfx_Character_DamageTakenWeakSpot_nl_meta.uasset
│   │   ├── sfx_Character_DamageTaken_nl_meta.uasset
│   │   ├── sfx_Character_FS_Base_nl_meta.uasset
│   │   ├── sfx_Character_FS_Concrete_nl_meta.uasset
│   │   ├── sfx_Character_FS_Glass_nl_meta.uasset
│   │   ├── sfx_Character_Land_Concrete_nl_meta.uasset
│   │   ├── sfx_Character_Land_Glass_nl_meta.uasset
│   │   ├── sfx_Character_RespawnTimer_meta.uasset
│   │   ├── sfx_DashStereo_nl_meta_Preset.uasset
│   │   ├── sfx_Death_nl_meta_Preset.uasset
│   │   ├── sfx_Emote_FingerGuns_nl_meta_Preset.uasset
│   │   ├── sfx_EndOfRound-LowScore_nl_meta.uasset
│   │   ├── sfx_EndOfRound-Tick_nl_meta.uasset
│   │   ├── sfx_Heal_nl_metaPreset.uasset
│   │   ├── sfx_ImpactCharacter_nl_meta_Preset.uasset
│   │   ├── sfx_ImpactDefault_nl_meta_Preset.uasset
│   │   ├── sfx_ImpactGlass_nl_meta_Preset.uasset
│   │   ├── sfx_ImpactPlaster_nl_meta.uasset
│   │   ├── sfx_Interactable_JumpPad_nl_meta.uasset
│   │   ├── sfx_Interactable_WeaponPad_Collect_nl_metaPreset.uasset
│   │   ├── sfx_Interactable_WeaponPad_Replenish_nl_metaPreset.uasset
│   │   ├── sfx_InventoryPad_lp_meta.uasset
│   │   ├── sfx_KillingSpree_nl_meta_Preset.uasset
│   │   ├── sfx_MeleeWhoosh_nl_meta_Preset.uasset
│   │   ├── sfx_RandomStereo_nl_meta.uasset
│   │   ├── sfx_RandomWLimitedRangeLayer_nl_meta.uasset
│   │   ├── sfx_Random_nl_meta.uasset
│   │   ├── sfx_Spawn_nl_meta_Preset.uasset
│   │   ├── sfx_Teleport_nl_meta_Preset.uasset
│   │   ├── sfx_WeaponSwap_nl_meta_Preset.uasset
│   │   ├── sfx_Weapon_BaseImpact_nl_metaPreset.uasset
│   │   ├── sfx_Weapon_FullyAutomatic_lp_meta.uasset
│   │   ├── sfx_Weapon_GrenadeExplosion_nl_meta.uasset
│   │   ├── sfx_Weapon_GrenadeImpact_nl_meta.uasset
│   │   ├── sfx_Weapon_MeleeImpact_nl_meta.uasset
│   │   ├── sfx_Weapon_Melee_nl_metaPreset.uasset
│   │   ├── sfx_Weapon_SemiAutomaticPistol_nl_metaPreset.uasset
│   │   ├── sfx_Weapon_SemiAutomaticRifle_nl_metaPreset.uasset
│   │   ├── sfx_Weapon_SemiAutomaticShotgun_nl_metaPreset.uasset
│   │   ├── sfx_Weapon_SemiAutomatic_nl_meta.uasset
│   │   ├── sfx_WhizBys_nl_meta.uasset
│   │   ├── sfx_ZoomIn_nl_meta_Preset.uasset
│   │   └── sfx_ZoomOut_nl_meta_Preset.uasset
│   ├── Modulation/
│   │   ├── ControlBusMixes/
│   │   │   ├── CBM_BaseMix.uasset
│   │   │   ├── CBM_InGameMenuMix.uasset
│   │   │   ├── CBM_LoadingScreenMix.uasset
│   │   │   └── CBM_UserMix.uasset
│   │   ├── ControlBuses/
│   │   │   ├── CB_Amb.uasset
│   │   │   ├── CB_Cinematics.uasset
│   │   │   ├── CB_Dialogue.uasset
│   │   │   ├── CB_Foley.uasset
│   │   │   ├── CB_Main.uasset
│   │   │   ├── CB_Music.uasset
│   │   │   ├── CB_SFX.uasset
│   │   │   ├── CB_UI.uasset
│   │   │   ├── CB_VoiceChat.uasset
│   │   │   └── CB_Weapons.uasset
│   │   └── ParameterPatches/
│   │       ├── PP_DefaultCinematics.uasset
│   │       ├── PP_DefaultDialogue.uasset
│   │       ├── PP_DefaultMusic.uasset
│   │       ├── PP_DefaultOverall.uasset
│   │       ├── PP_DefaultSFX.uasset
│   │       ├── PP_DefaultUI.uasset
│   │       └── PP_DefaultVoiceChat.uasset
│   ├── SoundWaves/
│   │   ├── Foley/
│   │   │   ├── SFX_Footstep_BaseLayer_01.uasset
│   │   │   ├── SFX_Footstep_BaseLayer_02.uasset
│   │   │   ├── SFX_Footstep_BaseLayer_03.uasset
│   │   │   ├── SFX_Footstep_BaseLayer_04.uasset
│   │   │   ├── SFX_Footstep_BaseLayer_05.uasset
│   │   │   ├── SFX_Footstep_BaseLayer_06.uasset
│   │   │   ├── SFX_Footstep_BaseLayer_07.uasset
│   │   │   ├── SFX_Footstep_BaseLayer_08.uasset
│   │   │   ├── SFX_Footstep_BaseLayer_09.uasset
│   │   │   ├── SFX_Footstep_BaseLayer_10.uasset
│   │   │   └── SFX_Footstep_BaseLayer_11.uasset
│   │   ├── SFX/
│   │   │   └── Text_Pop.uasset
│   │   └── Weapons/
│   │       ├── SFX_BulletImpact_01.uasset
│   │       ├── SFX_BulletImpact_02.uasset
│   │       ├── SFX_BulletImpact_03.uasset
│   │       ├── SFX_BulletImpact_04.uasset
│   │       ├── SFX_BulletImpact_05.uasset
│   │       ├── SFX_BulletImpact_06.uasset
│   │       ├── SFX_BulletImpact_07.uasset
│   │       ├── SFX_BulletImpact_08.uasset
│   │       ├── sfx_Weapon_AutoRifle_AutoClick_01.uasset
│   │       ├── sfx_Weapon_AutoRifle_AutoClick_02.uasset
│   │       ├── sfx_Weapon_AutoRifle_AutoClick_03.uasset
│   │       ├── sfx_Weapon_AutoRifle_AutoClick_04.uasset
│   │       ├── sfx_Weapon_AutoRifle_DryClick_01.uasset
│   │       ├── sfx_Weapon_AutoRifle_DryClick_02.uasset
│   │       ├── sfx_Weapon_AutoRifle_DryClick_03.uasset
│   │       ├── sfx_Weapon_AutoRifle_DryClick_04.uasset
│   │       ├── sfx_Weapon_AutoRifle_MainLayer_01.uasset
│   │       ├── sfx_Weapon_AutoRifle_MainLayer_02.uasset
│   │       ├── sfx_Weapon_AutoRifle_MainLayer_03.uasset
│   │       ├── sfx_Weapon_AutoRifle_MainLayer_04.uasset
│   │       ├── sfx_Weapon_AutoRifle_MainTail_01.uasset
│   │       ├── sfx_Weapon_AutoRifle_MainTail_02.uasset
│   │       ├── sfx_Weapon_AutoRifle_MainTail_03.uasset
│   │       ├── sfx_Weapon_AutoRifle_MainTail_04.uasset
│   │       ├── sfx_Weapon_AutoRifle_ShotSweetener_01.uasset
│   │       ├── sfx_Weapon_AutoRifle_ShotSweetener_02.uasset
│   │       ├── sfx_Weapon_AutoRifle_ShotSweetener_03.uasset
│   │       ├── sfx_Weapon_AutoRifle_ShotSweetener_04.uasset
│   │       ├── sfx_Weapon_Pistol_DryClick_nl_01.uasset
│   │       ├── sfx_Weapon_Pistol_DryClick_nl_02.uasset
│   │       ├── sfx_Weapon_Pistol_DryClick_nl_03.uasset
│   │       ├── sfx_Weapon_Pistol_DryClick_nl_04.uasset
│   │       ├── sfx_Weapon_Pistol_DryClick_nl_05.uasset
│   │       ├── sfx_Weapon_Pistol_MainLayer_nl_01.uasset
│   │       ├── sfx_Weapon_Pistol_MainLayer_nl_02.uasset
│   │       ├── sfx_Weapon_Pistol_MainLayer_nl_03.uasset
│   │       ├── sfx_Weapon_Pistol_MainLayer_nl_04.uasset
│   │       ├── sfx_Weapon_Pistol_MainLayer_nl_05.uasset
│   │       ├── sfx_Weapon_Pistol_MainLayer_nl_06.uasset
│   │       ├── sfx_Weapon_Pistol_MainLayer_nl_07.uasset
│   │       ├── sfx_Weapon_Pistol_SecondaryLayer_nl_01.uasset
│   │       ├── sfx_Weapon_Pistol_SecondaryLayer_nl_02.uasset
│   │       ├── sfx_Weapon_Pistol_SecondaryLayer_nl_03.uasset
│   │       ├── sfx_Weapon_Pistol_SecondaryLayer_nl_04.uasset
│   │       ├── sfx_Weapon_Shotgun_AutoLoad_nl_01.uasset
│   │       ├── sfx_Weapon_Shotgun_AutoLoad_nl_02.uasset
│   │       ├── sfx_Weapon_Shotgun_AutoLoad_nl_03.uasset
│   │       ├── sfx_Weapon_Shotgun_AutoLoad_nl_04.uasset
│   │       ├── sfx_Weapon_Shotgun_AutoLoad_nl_05.uasset
│   │       ├── sfx_Weapon_Shotgun_DryClick_nl_01.uasset
│   │       ├── sfx_Weapon_Shotgun_DryClick_nl_02.uasset
│   │       ├── sfx_Weapon_Shotgun_DryClick_nl_03.uasset
│   │       ├── sfx_Weapon_Shotgun_DryClick_nl_04.uasset
│   │       ├── sfx_Weapon_Shotgun_DryClick_nl_05.uasset
│   │       ├── sfx_Weapon_Shotgun_MainLayer_nl_01.uasset
│   │       ├── sfx_Weapon_Shotgun_MainLayer_nl_02.uasset
│   │       ├── sfx_Weapon_Shotgun_MainLayer_nl_03.uasset
│   │       ├── sfx_Weapon_Shotgun_MainLayer_nl_04.uasset
│   │       ├── sfx_Weapon_Shotgun_MainLayer_nl_05.uasset
│   │       ├── sfx_Weapon_Shotgun_MainLayer_nl_06.uasset
│   │       ├── sfx_Weapon_Shotgun_SecondaryLayer_nl_01.uasset
│   │       ├── sfx_Weapon_Shotgun_SecondaryLayer_nl_02.uasset
│   │       ├── sfx_Weapon_Shotgun_SecondaryLayer_nl_03.uasset
│   │       ├── sfx_Weapon_Shotgun_SecondaryLayer_nl_04.uasset
│   │       └── sfx_Weapon_Shotgun_SecondaryLayer_nl_05.uasset
│   ├── Sounds/
│   │   ├── Ambience/
│   │   │   ├── Lyra_RoomTone_01.uasset
│   │   │   ├── Lyra_RoomTone_02.uasset
│   │   │   └── Lyra_RoomTone_03.uasset
│   │   ├── EditorUtilities/
│   │   │   └── UB_SetSoundWavesToBink.uasset
│   │   ├── Emotes/
│   │   │   ├── ANS_EmoteSound.uasset
│   │   │   ├── BI_EmoteSoundInterface.uasset
│   │   │   ├── Emote_FingerGuns_01.uasset
│   │   │   ├── Emote_FingerGuns_02.uasset
│   │   │   └── Emote_FingerGuns_03.uasset
│   │   ├── Explosions/
│   │   │   ├── Explosions_Grenade_Noise-Exterior-Distant_01.uasset
│   │   │   ├── Explosions_Grenade_Noise-Exterior-Distant_02.uasset
│   │   │   ├── Explosions_Grenade_Noise-Exterior-Distant_03.uasset
│   │   │   ├── Explosions_Grenade_Noise-Exterior-Distant_04.uasset
│   │   │   ├── Explosions_Grenade_Noise-Exterior-Distant_05.uasset
│   │   │   ├── Explosions_Grenade_Noise-Exterior-Distant_06.uasset
│   │   │   ├── Explosions_Grenade_Noise-Exterior-Distant_07.uasset
│   │   │   ├── Explosions_Grenade_Noise-Exterior-Distant_08.uasset
│   │   │   ├── Explosions_Grenade_Noise-Exterior_01.uasset
│   │   │   ├── Explosions_Grenade_Noise-Exterior_02.uasset
│   │   │   ├── Explosions_Grenade_Noise-Exterior_03.uasset
│   │   │   ├── Explosions_Grenade_Noise-Exterior_04.uasset
│   │   │   ├── Explosions_Grenade_Noise-Interior-Distant_01.uasset
│   │   │   ├── Explosions_Grenade_Noise-Interior-Distant_02.uasset
│   │   │   ├── Explosions_Grenade_Noise-Interior-Distant_03.uasset
│   │   │   ├── Explosions_Grenade_Noise-Interior-Distant_04.uasset
│   │   │   ├── Explosions_Grenade_Noise-Interior_01.uasset
│   │   │   ├── Explosions_Grenade_Noise-Interior_02.uasset
│   │   │   ├── Explosions_Grenade_Noise-Interior_03.uasset
│   │   │   ├── Explosions_Grenade_Noise-Interior_04.uasset
│   │   │   ├── Explosions_Grenade_Punch-Close_01.uasset
│   │   │   ├── Explosions_Grenade_Punch-Close_02.uasset
│   │   │   ├── Explosions_Grenade_Punch-Close_03.uasset
│   │   │   ├── Explosions_Grenade_Punch-Close_04.uasset
│   │   │   ├── Explosions_Grenade_Punch-Close_05.uasset
│   │   │   ├── Explosions_Grenade_Punch-Close_06.uasset
│   │   │   ├── Explosions_Grenade_Punch-Distant_01.uasset
│   │   │   ├── Explosions_Grenade_Punch-Distant_02.uasset
│   │   │   ├── Explosions_Grenade_Punch-Distant_03.uasset
│   │   │   ├── Explosions_Grenade_Punch-Distant_04.uasset
│   │   │   ├── Explosions_Grenade_Ricochet_01.uasset
│   │   │   ├── Explosions_Grenade_Ricochet_02.uasset
│   │   │   ├── Explosions_Grenade_Ricochet_03.uasset
│   │   │   ├── Explosions_Grenade_Ricochet_04.uasset
│   │   │   ├── Explosions_Grenade_Ricochet_05.uasset
│   │   │   ├── Explosions_Grenade_Ricochet_06.uasset
│   │   │   ├── Explosions_Grenade_Ricochet_07.uasset
│   │   │   ├── Explosions_Grenade_Ricochet_08.uasset
│   │   │   ├── Explosions_Grenade_SFX_01.uasset
│   │   │   ├── Explosions_Grenade_SFX_02.uasset
│   │   │   ├── Explosions_Grenade_SFX_03.uasset
│   │   │   ├── Explosions_Grenade_SFX_04.uasset
│   │   │   ├── Explosions_Grenade_SFX_05.uasset
│   │   │   ├── Explosions_Grenade_SFX_06.uasset
│   │   │   ├── Explosions_Grenade_SFX_07.uasset
│   │   │   ├── Explosions_Grenade_SFX_08.uasset
│   │   │   ├── Explosions_Grenade_SFX_09.uasset
│   │   │   ├── Explosions_Grenade_SFX_10.uasset
│   │   │   ├── Explosions_Grenade_SFX_11.uasset
│   │   │   ├── Explosions_Grenade_SFX_12.uasset
│   │   │   ├── Explosions_Grenade_SFX_13.uasset
│   │   │   ├── Explosions_Grenade_SFX_14.uasset
│   │   │   └── MSS_Explosions_Grenade.uasset
│   │   ├── Foley_Misc/
│   │   │   ├── Lyra_FoleyLoop_01.uasset
│   │   │   ├── Lyra_FoleyStep_01.uasset
│   │   │   ├── Lyra_FoleyStep_02.uasset
│   │   │   ├── Lyra_FoleyStep_03.uasset
│   │   │   ├── Lyra_FoleyStep_04.uasset
│   │   │   ├── Lyra_FoleyStep_05.uasset
│   │   │   ├── Lyra_FoleyStep_06.uasset
│   │   │   ├── Lyra_FoleyStep_07.uasset
│   │   │   ├── Lyra_FoleyStep_08.uasset
│   │   │   ├── Lyra_FoleyStep_09.uasset
│   │   │   ├── Lyra_WeaponSwap_01.uasset
│   │   │   ├── Lyra_WeaponSwap_02.uasset
│   │   │   ├── Lyra_WeaponSwap_03.uasset
│   │   │   ├── Lyra_ZoomIn_01.uasset
│   │   │   ├── Lyra_ZoomIn_02.uasset
│   │   │   ├── Lyra_ZoomIn_03.uasset
│   │   │   ├── Lyra_ZoomOut_01.uasset
│   │   │   ├── Lyra_ZoomOut_02.uasset
│   │   │   └── Lyra_ZoomOut_03.uasset
│   │   ├── Footsteps/
│   │   │   ├── Lyra_Plyr_Foot_Glass_L_01.uasset
│   │   │   ├── Lyra_Plyr_Foot_Glass_L_02.uasset
│   │   │   ├── Lyra_Plyr_Foot_Glass_L_03.uasset
│   │   │   ├── Lyra_Plyr_Foot_Glass_L_04.uasset
│   │   │   ├── Lyra_Plyr_Foot_Glass_R_01.uasset
│   │   │   ├── Lyra_Plyr_Foot_Glass_R_02.uasset
│   │   │   ├── Lyra_Plyr_Foot_Glass_R_03.uasset
│   │   │   ├── Lyra_Plyr_Foot_Glass_R_04.uasset
│   │   │   ├── Lyra_Plyr_Foot_L_01.uasset
│   │   │   ├── Lyra_Plyr_Foot_L_02.uasset
│   │   │   ├── Lyra_Plyr_Foot_L_03.uasset
│   │   │   ├── Lyra_Plyr_Foot_L_04.uasset
│   │   │   ├── Lyra_Plyr_Foot_L_05.uasset
│   │   │   ├── Lyra_Plyr_Foot_L_06.uasset
│   │   │   ├── Lyra_Plyr_Foot_R_01.uasset
│   │   │   ├── Lyra_Plyr_Foot_R_02.uasset
│   │   │   ├── Lyra_Plyr_Foot_R_03.uasset
│   │   │   ├── Lyra_Plyr_Foot_R_04.uasset
│   │   │   ├── Lyra_Plyr_Foot_R_05.uasset
│   │   │   ├── Lyra_Plyr_Foot_R_06.uasset
│   │   │   ├── Lyra_Plyr_Land_Generic_01.uasset
│   │   │   ├── Lyra_Plyr_Land_Generic_02.uasset
│   │   │   ├── Lyra_Plyr_Land_Generic_03.uasset
│   │   │   ├── Lyra_Plyr_Land_Generic_04.uasset
│   │   │   ├── Lyra_Plyr_Land_Glass_01.uasset
│   │   │   ├── Lyra_Plyr_Land_Glass_02.uasset
│   │   │   ├── Lyra_Plyr_Land_Glass_03.uasset
│   │   │   └── Lyra_Plyr_Land_Glass_04.uasset
│   │   ├── Grenade/
│   │   │   ├── grenade_bounce_01.uasset
│   │   │   ├── grenade_bounce_02.uasset
│   │   │   ├── grenade_bounce_03.uasset
│   │   │   ├── grenade_bounce_04.uasset
│   │   │   ├── grenade_bounce_05.uasset
│   │   │   ├── grenade_explode_body_01.uasset
│   │   │   ├── grenade_explode_body_02.uasset
│   │   │   ├── grenade_explode_body_03.uasset
│   │   │   ├── grenade_explode_sizzle_01.uasset
│   │   │   ├── grenade_explode_sizzle_02.uasset
│   │   │   ├── grenade_explode_thump_01.uasset
│   │   │   ├── grenade_explode_thump_02.uasset
│   │   │   ├── grenade_explode_thump_03.uasset
│   │   │   ├── grenade_explode_whack_01.uasset
│   │   │   ├── grenade_explode_whack_02.uasset
│   │   │   ├── grenade_explode_whack_03.uasset
│   │   │   ├── grenade_explode_whack_04.uasset
│   │   │   └── grenade_explode_whack_05.uasset
│   │   ├── Impacts/
│   │   │   ├── Lyra_EnemyDestroyed_01.uasset
│   │   │   ├── Lyra_EnemyDestroyed_02.uasset
│   │   │   ├── Lyra_EnemyDestroyed_03.uasset
│   │   │   ├── Lyra_EnemyDestroyed_04.uasset
│   │   │   ├── Lyra_EnemyDestroyed_05.uasset
│   │   │   ├── Lyra_EnemyDestroyed_06.uasset
│   │   │   ├── Lyra_EnemyDestroyed_07.uasset
│   │   │   ├── Lyra_EnemyKilled_01.uasset
│   │   │   ├── Lyra_EnemyKilled_02.uasset
│   │   │   ├── Lyra_EnemyKilled_03.uasset
│   │   │   ├── Lyra_EnemyKilled_04.uasset
│   │   │   ├── Lyra_EnemyKilled_05.uasset
│   │   │   ├── Lyra_ImpactGlassDebris_01.uasset
│   │   │   ├── Lyra_ImpactGlassDebris_02.uasset
│   │   │   ├── Lyra_ImpactGlassDebris_03.uasset
│   │   │   ├── Lyra_ImpactGlassDebris_04.uasset
│   │   │   ├── Lyra_ImpactGlassDebris_05.uasset
│   │   │   ├── Lyra_ImpactGlassDebris_06.uasset
│   │   │   ├── Lyra_ImpactGlassDebris_07.uasset
│   │   │   ├── Lyra_ImpactGlass_01.uasset
│   │   │   ├── Lyra_ImpactGlass_02.uasset
│   │   │   ├── Lyra_ImpactGlass_03.uasset
│   │   │   ├── Lyra_ImpactGlass_04.uasset
│   │   │   ├── Lyra_ImpactGlass_05.uasset
│   │   │   ├── Lyra_ImpactGlass_06.uasset
│   │   │   ├── Lyra_ImpactGlass_07.uasset
│   │   │   ├── Lyra_ImpactHeadshot_01.uasset
│   │   │   ├── Lyra_ImpactHeadshot_02.uasset
│   │   │   ├── Lyra_ImpactHeadshot_03.uasset
│   │   │   ├── Lyra_ImpactHeadshot_04.uasset
│   │   │   ├── Lyra_ImpactHeadshot_05.uasset
│   │   │   ├── Lyra_ImpactHeadshot_06.uasset
│   │   │   ├── Lyra_ImpactPlasterDebris_01.uasset
│   │   │   ├── Lyra_ImpactPlasterDebris_02.uasset
│   │   │   ├── Lyra_ImpactPlasterDebris_03.uasset
│   │   │   ├── Lyra_ImpactPlasterDebris_04.uasset
│   │   │   ├── Lyra_ImpactPlasterDebris_05.uasset
│   │   │   ├── Lyra_ImpactPlasterDebris_06.uasset
│   │   │   ├── Lyra_ImpactPlasterDebris_07.uasset
│   │   │   ├── Lyra_ImpactPlaster_01.uasset
│   │   │   ├── Lyra_ImpactPlaster_02.uasset
│   │   │   ├── Lyra_ImpactPlaster_03.uasset
│   │   │   ├── Lyra_ImpactPlaster_04.uasset
│   │   │   ├── Lyra_ImpactPlaster_05.uasset
│   │   │   ├── Lyra_ImpactPlaster_06.uasset
│   │   │   ├── Lyra_ImpactPlaster_07.uasset
│   │   │   ├── Lyra_ImpactPlaster_08.uasset
│   │   │   ├── Lyra_ImpactPlaster_09.uasset
│   │   │   ├── Lyra_ImpactPlaster_10.uasset
│   │   │   ├── Lyra_Plyr_BulletImpact_01.uasset
│   │   │   ├── Lyra_Plyr_BulletImpact_02.uasset
│   │   │   ├── Lyra_Plyr_BulletImpact_03.uasset
│   │   │   ├── Lyra_Plyr_BulletImpact_04.uasset
│   │   │   ├── Lyra_Plyr_BulletImpact_05.uasset
│   │   │   └── Lyra_Plyr_BulletImpact_06.uasset
│   │   ├── Movement/
│   │   │   ├── Movement_Foley-Loop_01.uasset
│   │   │   ├── Movement_Foley-Step_01.uasset
│   │   │   ├── Movement_Foley-Step_02.uasset
│   │   │   ├── Movement_Foley-Step_03.uasset
│   │   │   ├── Movement_Foley-Step_04.uasset
│   │   │   ├── Movement_Foley-Step_05.uasset
│   │   │   ├── Movement_Foley-Step_06.uasset
│   │   │   ├── Movement_Foley-Step_07.uasset
│   │   │   ├── Movement_Foley-Step_08.uasset
│   │   │   ├── Movement_Foley-Step_09.uasset
│   │   │   ├── Movement_Footstep-Default_L_01.uasset
│   │   │   ├── Movement_Footstep-Default_L_02.uasset
│   │   │   ├── Movement_Footstep-Default_L_03.uasset
│   │   │   ├── Movement_Footstep-Default_L_04.uasset
│   │   │   ├── Movement_Footstep-Default_L_05.uasset
│   │   │   ├── Movement_Footstep-Default_L_06.uasset
│   │   │   ├── Movement_Footstep-Default_R_01.uasset
│   │   │   ├── Movement_Footstep-Default_R_02.uasset
│   │   │   ├── Movement_Footstep-Default_R_03.uasset
│   │   │   ├── Movement_Footstep-Default_R_04.uasset
│   │   │   ├── Movement_Footstep-Default_R_05.uasset
│   │   │   ├── Movement_Footstep-Default_R_06.uasset
│   │   │   ├── Movement_Footstep-Glass_L_01.uasset
│   │   │   ├── Movement_Footstep-Glass_L_02.uasset
│   │   │   ├── Movement_Footstep-Glass_L_03.uasset
│   │   │   ├── Movement_Footstep-Glass_L_04.uasset
│   │   │   ├── Movement_Footstep-Glass_R_01.uasset
│   │   │   ├── Movement_Footstep-Glass_R_02.uasset
│   │   │   ├── Movement_Footstep-Glass_R_03.uasset
│   │   │   ├── Movement_Footstep-Glass_R_04.uasset
│   │   │   ├── Movement_Land-Generic_01.uasset
│   │   │   ├── Movement_Land-Generic_02.uasset
│   │   │   ├── Movement_Land-Generic_03.uasset
│   │   │   ├── Movement_Land-Generic_04.uasset
│   │   │   ├── Movement_Land-Glass_01.uasset
│   │   │   ├── Movement_Land-Glass_02.uasset
│   │   │   ├── Movement_Land-Glass_03.uasset
│   │   │   └── Movement_Land-Glass_04.uasset
│   │   ├── Music/
│   │   │   ├── mx_menu_perc-deep_01.uasset
│   │   │   ├── mx_menu_perc-deep_02.uasset
│   │   │   ├── mx_menu_perc-deep_03.uasset
│   │   │   ├── mx_menu_perc-deep_04.uasset
│   │   │   ├── mx_menu_perc-deep_05.uasset
│   │   │   ├── mx_menu_perc-deep_06.uasset
│   │   │   ├── mx_menu_perc-deep_07.uasset
│   │   │   ├── mx_menu_perc-light_01.uasset
│   │   │   ├── mx_menu_perc-light_02.uasset
│   │   │   ├── mx_menu_perc-light_03.uasset
│   │   │   ├── mx_menu_perc-light_04.uasset
│   │   │   ├── mx_menu_perc-light_05.uasset
│   │   │   ├── mx_menu_plips_01.uasset
│   │   │   ├── mx_menu_saw-bass_01.uasset
│   │   │   ├── mx_menu_short-pad_01.uasset
│   │   │   ├── mx_menu_short-pad_02.uasset
│   │   │   ├── mx_menu_wet-lead-a_01.uasset
│   │   │   ├── mx_menu_wet-lead-b_01.uasset
│   │   │   ├── mx_v1_amb_chord-a_long-pad_01.uasset
│   │   │   ├── mx_v1_amb_chord-a_short-pad_01.uasset
│   │   │   ├── mx_v1_amb_chord-b_long-pad_01.uasset
│   │   │   ├── mx_v1_amb_chord-b_short-pad_01.uasset
│   │   │   ├── mx_v1_amb_chord-c_long-pad_01.uasset
│   │   │   ├── mx_v1_amb_chord-c_short-pad_01.uasset
│   │   │   ├── mx_v1_amb_chord-d_long-pad_01.uasset
│   │   │   ├── mx_v1_amb_chord-d_short-pad_01.uasset
│   │   │   ├── mx_v1_amb_chord-e_long-pad_01.uasset
│   │   │   ├── mx_v1_amb_chord-e_short-pad_01.uasset
│   │   │   ├── mx_v1_combat_a_long-pad_01.uasset
│   │   │   ├── mx_v1_combat_a_long-pad_02.uasset
│   │   │   ├── mx_v1_combat_a_long-pad_03.uasset
│   │   │   ├── mx_v1_combat_a_long-pad_04.uasset
│   │   │   ├── mx_v1_combat_a_long-pad_05.uasset
│   │   │   ├── mx_v1_combat_a_long-pad_06.uasset
│   │   │   ├── mx_v1_combat_a_perc-deep_01.uasset
│   │   │   ├── mx_v1_combat_a_perc-deep_02.uasset
│   │   │   ├── mx_v1_combat_a_perc-deep_03.uasset
│   │   │   ├── mx_v1_combat_a_perc-deep_04.uasset
│   │   │   ├── mx_v1_combat_a_perc-deep_05.uasset
│   │   │   ├── mx_v1_combat_a_perc-deep_06.uasset
│   │   │   ├── mx_v1_combat_a_perc-deep_07.uasset
│   │   │   ├── mx_v1_combat_a_perc-deep_08.uasset
│   │   │   ├── mx_v1_combat_a_perc-deep_09.uasset
│   │   │   ├── mx_v1_combat_a_perc-deep_10.uasset
│   │   │   ├── mx_v1_combat_a_perc-deep_11.uasset
│   │   │   ├── mx_v1_combat_a_perc-deep_12.uasset
│   │   │   ├── mx_v1_combat_a_perc-deep_13.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_01.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_02.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_03.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_04.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_05.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_06.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_07.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_08.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_09.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_10.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_11.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_12.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_13.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_14.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_15.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_16.uasset
│   │   │   ├── mx_v1_combat_a_perc-light_17.uasset
│   │   │   ├── mx_v1_combat_a_plips_01.uasset
│   │   │   ├── mx_v1_combat_a_plips_02.uasset
│   │   │   ├── mx_v1_combat_a_plips_03.uasset
│   │   │   ├── mx_v1_combat_a_plips_04.uasset
│   │   │   ├── mx_v1_combat_a_plips_05.uasset
│   │   │   ├── mx_v1_combat_a_saw-bass_01.uasset
│   │   │   ├── mx_v1_combat_a_saw-bass_02.uasset
│   │   │   ├── mx_v1_combat_a_saw-bass_03.uasset
│   │   │   ├── mx_v1_combat_a_saw-bass_04.uasset
│   │   │   ├── mx_v1_combat_a_saw-bass_05.uasset
│   │   │   ├── mx_v1_combat_a_saw-bass_06.uasset
│   │   │   ├── mx_v1_combat_a_short-pad_01.uasset
│   │   │   ├── mx_v1_combat_a_short-pad_02.uasset
│   │   │   ├── mx_v1_combat_a_short-pad_03.uasset
│   │   │   ├── mx_v1_combat_a_short-pad_04.uasset
│   │   │   ├── mx_v1_combat_a_short-pad_05.uasset
│   │   │   ├── mx_v1_combat_a_short-pad_06.uasset
│   │   │   ├── mx_v1_combat_a_short-pad_07.uasset
│   │   │   ├── mx_v1_combat_a_short-pad_08.uasset
│   │   │   ├── mx_v1_combat_a_wet-lead_01.uasset
│   │   │   ├── mx_v1_combat_a_wet-lead_02.uasset
│   │   │   ├── mx_v1_combat_a_wet-lead_03.uasset
│   │   │   ├── mx_v1_combat_a_woo-bass_01.uasset
│   │   │   ├── mx_v1_combat_a_woo-bass_02.uasset
│   │   │   ├── mx_v1_combat_a_woo-bass_03.uasset
│   │   │   ├── mx_v1_combat_a_woo-bass_04.uasset
│   │   │   ├── mx_v1_combat_a_woo-bass_05.uasset
│   │   │   ├── mx_v1_combat_b_chord-a_dry-lead_01.uasset
│   │   │   ├── mx_v1_combat_b_chord-a_wet-lead_01.uasset
│   │   │   ├── mx_v1_combat_b_chord-a_woo-bass_01.uasset
│   │   │   ├── mx_v1_combat_b_chord-b_dry-lead_01.uasset
│   │   │   ├── mx_v1_combat_b_chord-b_wet-lead_01.uasset
│   │   │   ├── mx_v1_combat_b_chord-b_woo-bass_01.uasset
│   │   │   ├── mx_v1_combat_b_chord-c_dry-lead_01.uasset
│   │   │   ├── mx_v1_combat_b_chord-c_wet-lead_01.uasset
│   │   │   ├── mx_v1_combat_b_chord-c_woo-bass_01.uasset
│   │   │   ├── mx_v1_combat_b_chord-d_dry-lead_01.uasset
│   │   │   ├── mx_v1_combat_b_chord-d_wet-lead_01.uasset
│   │   │   ├── mx_v1_combat_b_chord-d_woo-bass_01.uasset
│   │   │   ├── mx_v1_combat_b_perc-deep_01.uasset
│   │   │   ├── mx_v1_combat_b_perc-deep_02.uasset
│   │   │   ├── mx_v1_combat_b_perc-deep_03.uasset
│   │   │   ├── mx_v1_combat_b_perc-deep_04.uasset
│   │   │   ├── mx_v1_combat_b_perc-deep_05.uasset
│   │   │   ├── mx_v1_combat_b_perc-deep_06.uasset
│   │   │   ├── mx_v1_combat_b_perc-deep_07.uasset
│   │   │   ├── mx_v1_combat_b_perc-deep_08.uasset
│   │   │   ├── mx_v1_combat_b_perc-light_01.uasset
│   │   │   ├── mx_v1_combat_b_perc-light_02.uasset
│   │   │   ├── mx_v1_combat_b_perc-light_03.uasset
│   │   │   ├── mx_v1_combat_b_perc-light_04.uasset
│   │   │   ├── mx_v1_combat_b_perc-light_05.uasset
│   │   │   ├── mx_v1_combat_b_perc-light_06.uasset
│   │   │   ├── mx_v1_combat_b_perc-light_07.uasset
│   │   │   ├── mx_v1_combat_b_perc-light_08.uasset
│   │   │   ├── mx_v1_combat_b_perc-light_09.uasset
│   │   │   ├── mx_v1_combat_b_perc-light_10.uasset
│   │   │   ├── mx_v1_combat_b_perc-light_11.uasset
│   │   │   ├── mx_v1_combat_b_perc-light_12.uasset
│   │   │   ├── mx_v1_combat_b_perc-light_13.uasset
│   │   │   ├── mx_v1_combat_b_plips_01.uasset
│   │   │   ├── mx_v1_combat_b_plips_02.uasset
│   │   │   ├── mx_v1_combat_b_plips_03.uasset
│   │   │   ├── mx_v1_combat_b_plips_04.uasset
│   │   │   ├── mx_v1_combat_b_short-pad_01.uasset
│   │   │   ├── mx_v1_combat_b_short-pad_02.uasset
│   │   │   ├── mx_v1_combat_b_short-pad_03.uasset
│   │   │   ├── mx_v1_combat_b_short-pad_04.uasset
│   │   │   ├── mx_v2_combat_a_perc-deep_01.uasset
│   │   │   ├── mx_v2_combat_a_perc-deep_02.uasset
│   │   │   ├── mx_v2_combat_a_perc-deep_03.uasset
│   │   │   ├── mx_v2_combat_a_perc-deep_04.uasset
│   │   │   ├── mx_v2_combat_a_perc-deep_05.uasset
│   │   │   ├── mx_v2_combat_a_perc-deep_06.uasset
│   │   │   ├── mx_v2_combat_a_perc-deep_07.uasset
│   │   │   ├── mx_v2_combat_a_perc-deep_08.uasset
│   │   │   ├── mx_v2_combat_a_perc-deep_09.uasset
│   │   │   ├── mx_v2_combat_a_perc-deep_10.uasset
│   │   │   ├── mx_v2_combat_a_perc-deep_11.uasset
│   │   │   ├── mx_v2_combat_a_perc-light_01.uasset
│   │   │   ├── mx_v2_combat_a_perc-light_02.uasset
│   │   │   ├── mx_v2_combat_a_perc-light_03.uasset
│   │   │   ├── mx_v2_combat_a_perc-light_04.uasset
│   │   │   ├── mx_v2_combat_a_perc-light_05.uasset
│   │   │   ├── mx_v2_combat_a_perc-light_06.uasset
│   │   │   ├── mx_v2_combat_a_perc-light_07.uasset
│   │   │   ├── mx_v2_combat_a_perc-light_08.uasset
│   │   │   ├── mx_v2_stinger_positive_perc-deep_01.uasset
│   │   │   ├── mx_v2_stinger_positive_perc-deep_02.uasset
│   │   │   ├── mx_v2_stinger_positive_perc-light_01.uasset
│   │   │   ├── mx_v2_stinger_positive_perc-light_02.uasset
│   │   │   ├── mx_v2_stinger_positive_saw-bass_01.uasset
│   │   │   ├── mx_v2_stinger_positive_saw-bass_02.uasset
│   │   │   ├── mx_v2_stinger_positive_short-pad_01.uasset
│   │   │   ├── mx_v2_stinger_positive_short-pad_02.uasset
│   │   │   ├── mx_v2_stinger_positive_wet-lead_01.uasset
│   │   │   ├── mx_v2_stinger_positive_wet-lead_02.uasset
│   │   │   ├── mx_v2_stinger_positive_woo-bass_01.uasset
│   │   │   └── mx_v2_stinger_positive_woo-bass_02.uasset
│   │   ├── SFX/
│   │   │   ├── Lyra_BasePickup_01.uasset
│   │   │   ├── Lyra_BasePickup_02.uasset
│   │   │   ├── Lyra_BasePickup_03.uasset
│   │   │   ├── Lyra_ControlPoint_CapturedByEnemy_01.uasset
│   │   │   ├── Lyra_ControlPoint_CapturedByFriendly_01.uasset
│   │   │   ├── Lyra_Dash_01.uasset
│   │   │   ├── Lyra_Dash_02.uasset
│   │   │   ├── Lyra_EndOfRound-Defeat_01.uasset
│   │   │   ├── Lyra_EndOfRound-Tick-01.uasset
│   │   │   ├── Lyra_EndOfRound-Victory_01.uasset
│   │   │   ├── Lyra_EndOfRound_01.uasset
│   │   │   ├── Lyra_ItemReplenish_01.uasset
│   │   │   ├── Lyra_ItemReplenish_02.uasset
│   │   │   ├── Lyra_JumpPad_01.uasset
│   │   │   ├── Lyra_JumpPad_02.uasset
│   │   │   ├── Lyra_JumpPad_03.uasset
│   │   │   ├── Lyra_JumpPad_Whoosh_01.uasset
│   │   │   ├── Lyra_JumpPad_Whoosh_02.uasset
│   │   │   ├── Lyra_KillingSpree_01.uasset
│   │   │   ├── Lyra_KillingSpree_02.uasset
│   │   │   ├── Lyra_KillingSpree_03.uasset
│   │   │   ├── Lyra_MeleeImpact_01.uasset
│   │   │   ├── Lyra_MeleeImpact_02.uasset
│   │   │   ├── Lyra_MeleeImpact_03.uasset
│   │   │   ├── Lyra_MeleeWhoosh_01.uasset
│   │   │   ├── Lyra_MeleeWhoosh_02.uasset
│   │   │   ├── Lyra_MeleeWhoosh_03.uasset
│   │   │   ├── Lyra_MeleeWhoosh_04.uasset
│   │   │   ├── Lyra_PickUpWeapon_01.uasset
│   │   │   ├── Lyra_Plyr_Spawn_01.uasset
│   │   │   ├── Lyra_Plyr_Spawn_02.uasset
│   │   │   ├── Lyra_Portal_01.uasset
│   │   │   ├── Lyra_Portal_02.uasset
│   │   │   ├── Lyra_Portal_03.uasset
│   │   │   ├── Lyra_Portal_04.uasset
│   │   │   ├── Lyra_Portal_05.uasset
│   │   │   ├── Lyra_RingLoop_01.uasset
│   │   │   ├── Lyra_RingLoop_02.uasset
│   │   │   ├── Lyra_RingLoop_03.uasset
│   │   │   └── SFX_Chime_03A.uasset
│   │   ├── UI/
│   │   │   ├── ButtonClick.uasset
│   │   │   ├── ButtonHover.uasset
│   │   │   ├── Lyra_UI_Lobby_Highlight_01.uasset
│   │   │   ├── Lyra_UI_Lobby_Highlight_02.uasset
│   │   │   ├── Lyra_UI_Lobby_Select.uasset
│   │   │   ├── Lyra_UI_MainMenu_Highlight_01.uasset
│   │   │   ├── Lyra_UI_MainMenu_Highlight_02.uasset
│   │   │   ├── Lyra_UI_MainMenu_Select.uasset
│   │   │   ├── Lyra_UI_SubMenu_Back.uasset
│   │   │   ├── Lyra_UI_SubMenu_Highlight_01.uasset
│   │   │   ├── Lyra_UI_SubMenu_Highlight_02.uasset
│   │   │   ├── sfx_UI_Lobby_Highlight_nl_meta_Preset.uasset
│   │   │   ├── sfx_UI_Lobby_Select_nl_meta_Preset.uasset
│   │   │   ├── sfx_UI_MainMenu_Highlight_nl_meta_Preset.uasset
│   │   │   ├── sfx_UI_MainMenu_Select_nl_meta_Preset.uasset
│   │   │   ├── sfx_UI_SubMenu_Back_nl_meta_Preset.uasset
│   │   │   ├── sfx_UI_SubMenu_Highlight_nl_meta_Preset.uasset
│   │   │   └── sfx_UI_SubMenu_Select_nl_meta_Preset.uasset
│   │   ├── Weapons/
│   │   │   ├── Pistol/
│   │   │   │   ├── MSS_Weapons_Pistol_Fire.uasset
│   │   │   │   ├── Weapons_Pistol_ClipIn_01.uasset
│   │   │   │   ├── Weapons_Pistol_ClipOut_01.uasset
│   │   │   │   ├── Weapons_Pistol_DryFire_01.uasset
│   │   │   │   ├── Weapons_Pistol_Mech_01.uasset
│   │   │   │   ├── Weapons_Pistol_Mech_02.uasset
│   │   │   │   ├── Weapons_Pistol_Mech_03.uasset
│   │   │   │   ├── Weapons_Pistol_Noise-Interior-Close_01.uasset
│   │   │   │   ├── Weapons_Pistol_Noise-Interior-Close_02.uasset
│   │   │   │   ├── Weapons_Pistol_Noise-Interior-Close_03.uasset
│   │   │   │   ├── Weapons_Pistol_Noise-Interior-Close_04.uasset
│   │   │   │   ├── Weapons_Pistol_Noise-Interior-Close_05.uasset
│   │   │   │   ├── Weapons_Pistol_Noise-Interior-Close_06.uasset
│   │   │   │   ├── Weapons_Pistol_Noise-Interior-Close_07.uasset
│   │   │   │   ├── Weapons_Pistol_Noise-Interior-Distant_01.uasset
│   │   │   │   ├── Weapons_Pistol_Noise-Interior-Distant_02.uasset
│   │   │   │   ├── Weapons_Pistol_Noise-Interior-Distant_03.uasset
│   │   │   │   ├── Weapons_Pistol_Noise-Interior-Distant_04.uasset
│   │   │   │   ├── Weapons_Pistol_Noise-Interior-Distant_05.uasset
│   │   │   │   ├── Weapons_Pistol_Punch-Close_01.uasset
│   │   │   │   ├── Weapons_Pistol_Punch-Close_02.uasset
│   │   │   │   ├── Weapons_Pistol_Punch-Close_03.uasset
│   │   │   │   ├── Weapons_Pistol_Punch-Close_04.uasset
│   │   │   │   ├── Weapons_Pistol_Punch-Distant_01.uasset
│   │   │   │   ├── Weapons_Pistol_Punch-Distant_02.uasset
│   │   │   │   ├── Weapons_Pistol_Punch-Distant_03.uasset
│   │   │   │   ├── Weapons_Pistol_Punch-Distant_04.uasset
│   │   │   │   ├── Weapons_Pistol_Punch-Far_01.uasset
│   │   │   │   ├── Weapons_Pistol_Punch-Far_02.uasset
│   │   │   │   ├── Weapons_Pistol_Punch-Far_03.uasset
│   │   │   │   ├── Weapons_Pistol_Punch-Far_04.uasset
│   │   │   │   ├── Weapons_Pistol_Punch-Far_05.uasset
│   │   │   │   ├── Weapons_Pistol_Slide_01.uasset
│   │   │   │   ├── Weapons_Pistol_Sweetener_01.uasset
│   │   │   │   ├── Weapons_Pistol_Sweetener_02.uasset
│   │   │   │   ├── Weapons_Pistol_Sweetener_03.uasset
│   │   │   │   └── Weapons_Pistol_Sweetener_04.uasset
│   │   │   ├── Rifle/
│   │   │   │   ├── MSS_Weapons_Rifle_Fire.uasset
│   │   │   │   ├── Weapons_Rifle_Bolt_01.uasset
│   │   │   │   ├── Weapons_Rifle_ClipIn_01.uasset
│   │   │   │   ├── Weapons_Rifle_ClipOut_01.uasset
│   │   │   │   ├── Weapons_Rifle_DryFire_01.uasset
│   │   │   │   ├── Weapons_Rifle_Mech_01.uasset
│   │   │   │   ├── Weapons_Rifle_Mech_02.uasset
│   │   │   │   ├── Weapons_Rifle_Mech_03.uasset
│   │   │   │   ├── Weapons_Rifle_Mech_04.uasset
│   │   │   │   ├── Weapons_Rifle_Mech_05.uasset
│   │   │   │   ├── Weapons_Rifle_Mech_06.uasset
│   │   │   │   ├── Weapons_Rifle_Mech_07.uasset
│   │   │   │   ├── Weapons_Rifle_Noise-Exterior-Close_01.uasset
│   │   │   │   ├── Weapons_Rifle_Noise-Exterior-Close_02.uasset
│   │   │   │   ├── Weapons_Rifle_Noise-Exterior-Close_03.uasset
│   │   │   │   ├── Weapons_Rifle_Noise-Exterior-Close_04.uasset
│   │   │   │   ├── Weapons_Rifle_Noise-Exterior-Close_05.uasset
│   │   │   │   ├── Weapons_Rifle_Noise-Exterior-Close_06.uasset
│   │   │   │   ├── Weapons_Rifle_Noise-Exterior-Distant_01.uasset
│   │   │   │   ├── Weapons_Rifle_Noise-Exterior-Distant_02.uasset
│   │   │   │   ├── Weapons_Rifle_Noise-Exterior-Distant_03.uasset
│   │   │   │   ├── Weapons_Rifle_Noise-Exterior-Distant_04.uasset
│   │   │   │   ├── Weapons_Rifle_Noise-Exterior-Distant_05.uasset
│   │   │   │   ├── Weapons_Rifle_Noise-Exterior-Distant_06.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_01.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_02.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_03.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_04.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_05.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_06.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_07.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_08.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_09.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_10.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_11.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_12.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_13.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_14.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_15.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_16.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_17.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_18.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_19.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_20.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_21.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Close_22.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_01.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_02.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_03.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_04.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_05.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_06.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_07.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_08.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_09.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_10.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_11.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_12.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_13.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_14.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_15.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Distant_16.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_01.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_02.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_03.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_04.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_05.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_06.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_07.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_08.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_09.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_10.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_11.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_12.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_13.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_14.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_15.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_16.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_17.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_18.uasset
│   │   │   │   ├── Weapons_Rifle_Punch-Far_19.uasset
│   │   │   │   └── Weapons_Rifle_Punch-Far_20.uasset
│   │   │   ├── Rifle2/
│   │   │   │   ├── MSS_Weapons_Rifle2_Fire.uasset
│   │   │   │   ├── Weapons_Rifle2_Mech_01.uasset
│   │   │   │   ├── Weapons_Rifle2_Mech_02.uasset
│   │   │   │   ├── Weapons_Rifle2_Mech_03.uasset
│   │   │   │   ├── Weapons_Rifle2_Mech_04.uasset
│   │   │   │   ├── Weapons_Rifle2_Mech_05.uasset
│   │   │   │   ├── Weapons_Rifle2_Mech_06.uasset
│   │   │   │   ├── Weapons_Rifle2_Mech_07.uasset
│   │   │   │   ├── Weapons_Rifle2_Mech_08.uasset
│   │   │   │   ├── Weapons_Rifle2_Mech_09.uasset
│   │   │   │   ├── Weapons_Rifle2_Mech_10.uasset
│   │   │   │   ├── Weapons_Rifle2_Mech_11.uasset
│   │   │   │   ├── Weapons_Rifle2_Mech_12.uasset
│   │   │   │   ├── Weapons_Rifle2_Mech_13.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Close_01.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Close_02.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Close_03.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Close_04.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Close_05.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Close_06.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Close_07.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Close_08.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Distant_01.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Distant_02.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Distant_03.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Distant_04.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Distant_05.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Distant_06.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Distant_07.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Distant_08.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Distant_09.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Distant_10.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Distant_11.uasset
│   │   │   │   ├── Weapons_Rifle2_Noise-Interior-Distant_12.uasset
│   │   │   │   ├── Weapons_Rifle2_Punch_01.uasset
│   │   │   │   ├── Weapons_Rifle2_Punch_02.uasset
│   │   │   │   ├── Weapons_Rifle2_Punch_03.uasset
│   │   │   │   ├── Weapons_Rifle2_Punch_04.uasset
│   │   │   │   ├── Weapons_Rifle2_Punch_05.uasset
│   │   │   │   ├── Weapons_Rifle2_Punch_06.uasset
│   │   │   │   ├── Weapons_Rifle2_Punch_07.uasset
│   │   │   │   ├── Weapons_Rifle2_Punch_08.uasset
│   │   │   │   ├── Weapons_Rifle2_Punch_09.uasset
│   │   │   │   ├── Weapons_Rifle2_Punch_10.uasset
│   │   │   │   ├── Weapons_Rifle2_Punch_11.uasset
│   │   │   │   ├── Weapons_Rifle2_Punch_12.uasset
│   │   │   │   ├── Weapons_Rifle2_Punch_13.uasset
│   │   │   │   ├── Weapons_Rifle2_Sweetener_01.uasset
│   │   │   │   ├── Weapons_Rifle2_Sweetener_02.uasset
│   │   │   │   ├── Weapons_Rifle2_Sweetener_03.uasset
│   │   │   │   └── Weapons_Rifle2_Sweetener_04.uasset
│   │   │   ├── Shotgun/
│   │   │   │   ├── MSS_Weapons_Shotgun_Fire.uasset
│   │   │   │   ├── Weapons_Shotgun_ClipIn_01.uasset
│   │   │   │   ├── Weapons_Shotgun_ClipOut_01.uasset
│   │   │   │   ├── Weapons_Shotgun_DryFire_01.uasset
│   │   │   │   ├── Weapons_Shotgun_HandHit_01.uasset
│   │   │   │   ├── Weapons_Shotgun_Mech_01.uasset
│   │   │   │   ├── Weapons_Shotgun_Mech_02.uasset
│   │   │   │   ├── Weapons_Shotgun_Mech_03.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Exterior-Close_01.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Exterior-Close_02.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Exterior-Close_03.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Exterior-Distant_01.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Exterior-Distant_02.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Exterior-Distant_03.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Exterior-Far_02.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Exterior-Far_03.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Interior-Close_01.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Interior-Close_02.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Interior-Close_03.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Interior-Close_04.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Interior-Distant_01.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Interior-Distant_02.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Interior-Distant_03.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Interior-Distant_04.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Interior-Distant_05.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Interior-Far_01.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Interior-Far_02.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Interior-Far_03.uasset
│   │   │   │   ├── Weapons_Shotgun_Noise-Interior-Far_04.uasset
│   │   │   │   ├── Weapons_Shotgun_Punch_01.uasset
│   │   │   │   ├── Weapons_Shotgun_Punch_02.uasset
│   │   │   │   ├── Weapons_Shotgun_Punch_03.uasset
│   │   │   │   ├── Weapons_Shotgun_Sweetener_01.uasset
│   │   │   │   ├── Weapons_Shotgun_Sweetener_02.uasset
│   │   │   │   └── Weapons_Shotgun_Sweetener_03.uasset
│   │   │   ├── MS_EqualPowerCrossfade.uasset
│   │   │   ├── MS_GatedWavePlayer.uasset
│   │   │   ├── MS_LowAmmoTone.uasset
│   │   │   ├── MS_RandomEQ.uasset
│   │   │   ├── MS_StereoGain.uasset
│   │   │   ├── MS_StereoHighShelf.uasset
│   │   │   ├── MS_WaveArrayCrossfader.uasset
│   │   │   └── MS_WavePlayerCrossfader.uasset
│   │   └── WhizBys/
│   │       ├── Lyra_BulletIn_Close_01.uasset
│   │       ├── Lyra_BulletIn_Close_02.uasset
│   │       ├── Lyra_BulletIn_Close_03.uasset
│   │       ├── Lyra_BulletIn_Close_04.uasset
│   │       ├── Lyra_BulletIn_Close_05.uasset
│   │       ├── Lyra_BulletIn_Close_06.uasset
│   │       ├── Lyra_BulletIn_Close_07.uasset
│   │       ├── Lyra_BulletIn_Close_08.uasset
│   │       ├── Lyra_BulletOut_Close_01.uasset
│   │       ├── Lyra_BulletOut_Close_02.uasset
│   │       ├── Lyra_BulletOut_Close_03.uasset
│   │       ├── Lyra_BulletOut_Close_04.uasset
│   │       ├── Lyra_BulletOut_Close_05.uasset
│   │       ├── Lyra_BulletOut_Close_06.uasset
│   │       ├── Lyra_BulletOut_Close_07.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Close_01.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Close_02.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Close_03.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Close_04.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Far_01.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Far_02.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Far_03.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Far_04.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Far_05.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Far_06.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Mid_01.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Mid_02.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Mid_03.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Mid_04.uasset
│   │       ├── sfx_AR_Whizbys_Oncoming_Mid_05.uasset
│   │       ├── sfx_AR_Whizbys_Receding_Close_01.uasset
│   │       ├── sfx_AR_Whizbys_Receding_Close_02.uasset
│   │       ├── sfx_AR_Whizbys_Receding_Close_03.uasset
│   │       ├── sfx_AR_Whizbys_Receding_Close_04.uasset
│   │       ├── sfx_AR_Whizbys_Receding_Far_01.uasset
│   │       ├── sfx_AR_Whizbys_Receding_Far_02.uasset
│   │       ├── sfx_AR_Whizbys_Receding_Far_03.uasset
│   │       ├── sfx_AR_Whizbys_Receding_Far_04.uasset
│   │       ├── sfx_AR_Whizbys_Receding_Far_05.uasset
│   │       ├── sfx_AR_Whizbys_Receding_Far_06.uasset
│   │       ├── sfx_AR_Whizbys_Receding_Mid_01.uasset
│   │       ├── sfx_AR_Whizbys_Receding_Mid_02.uasset
│   │       ├── sfx_AR_Whizbys_Receding_Mid_03.uasset
│   │       ├── sfx_AR_Whizbys_Receding_Mid_04.uasset
│   │       └── sfx_AR_Whizbys_Receding_Mid_05.uasset
│   ├── Submixes/
│   │   ├── EarlyReflectionsSubmix.uasset
│   │   ├── MainSubmix.uasset
│   │   ├── MusicSubmix.uasset
│   │   ├── ReverbSubmix.uasset
│   │   ├── SFXSubmix.uasset
│   │   ├── SendEffectSubmix.uasset
│   │   ├── UISubmix.uasset
│   │   └── VoiceSubmix.uasset
│   └── DYN_LowMultibandDynamics.uasset
├── Characters/
│   ├── Cameras/
│   │   ├── CM_ThirdPerson.uasset
│   │   ├── CM_ThirdPerson_Death.uasset
│   │   ├── ThirdPersonDeathOffsetCurve.uasset
│   │   └── ThirdPersonOffsetCurve.uasset
│   ├── Cosmetics/
│   │   ├── B_CharacterSelection.uasset
│   │   ├── B_MannequinPawnCosmetics.uasset
│   │   ├── B_Manny.uasset
│   │   ├── B_PickRandomCharacter.uasset
│   │   ├── B_Quinn.uasset
│   │   ├── B_TinplateUE4.uasset
│   │   └── M_TeamColorBasic.uasset
│   ├── Heroes/
│   │   ├── Abilities/
│   │   │   ├── AN_Melee.uasset
│   │   │   ├── AN_Reload.uasset
│   │   │   ├── GA_AbilityWithWidget.uasset
│   │   │   ├── GA_Hero_Death.uasset
│   │   │   ├── GA_Hero_Heal.uasset
│   │   │   ├── GA_Hero_Jump.uasset
│   │   │   ├── W_CrouchTouchButton.uasset
│   │   │   └── W_JumpTouchButton.uasset
│   │   ├── EmptyPawnData/
│   │   │   └── DefaultPawnData_EmptyPawn.uasset
│   │   ├── Mannequin/
│   │   │   ├── Animations/
│   │   │   │   ├── Actions/
│   │   │   │   │   ├── AM_MF_Emote_FingerGuns.uasset
│   │   │   │   │   ├── AM_MF_Emote_FingerGuns_Emote_MW.uasset
│   │   │   │   │   ├── AM_MM_Dash_Backward.uasset
│   │   │   │   │   ├── AM_MM_Dash_Forward.uasset
│   │   │   │   │   ├── AM_MM_Dash_Forward_LoadingScreenStills.uasset
│   │   │   │   │   ├── AM_MM_Dash_Left.uasset
│   │   │   │   │   ├── AM_MM_Dash_Right.uasset
│   │   │   │   │   ├── AM_MM_Death_Back_01.uasset
│   │   │   │   │   ├── AM_MM_Death_Front_01.uasset
│   │   │   │   │   ├── AM_MM_Death_Front_02.uasset
│   │   │   │   │   ├── AM_MM_Death_Front_03.uasset
│   │   │   │   │   ├── AM_MM_Death_Left_01.uasset
│   │   │   │   │   ├── AM_MM_Death_Right_01.uasset
│   │   │   │   │   ├── AM_MM_Generic_Unequip.uasset
│   │   │   │   │   ├── AM_MM_HitReact_Back_Lgt_01.uasset
│   │   │   │   │   ├── AM_MM_HitReact_Back_Med_01.uasset
│   │   │   │   │   ├── AM_MM_HitReact_Front_Hvy_01.uasset
│   │   │   │   │   ├── AM_MM_HitReact_Front_Lgt_01.uasset
│   │   │   │   │   ├── AM_MM_HitReact_Front_Lgt_02.uasset
│   │   │   │   │   ├── AM_MM_HitReact_Front_Lgt_03.uasset
│   │   │   │   │   ├── AM_MM_HitReact_Front_Lgt_04.uasset
│   │   │   │   │   ├── AM_MM_HitReact_Front_Med_01.uasset
│   │   │   │   │   ├── AM_MM_HitReact_Front_Med_02.uasset
│   │   │   │   │   ├── AM_MM_HitReact_Left_Lgt_01.uasset
│   │   │   │   │   ├── AM_MM_HitReact_Left_Med_01.uasset
│   │   │   │   │   ├── AM_MM_HitReact_Right_Lgt_01.uasset
│   │   │   │   │   ├── AM_MM_HitReact_Right_Med_01.uasset
│   │   │   │   │   ├── AM_MM_Pistol_Melee.uasset
│   │   │   │   │   ├── AM_MM_Pistol_Spawn.uasset
│   │   │   │   │   ├── AM_MM_Rifle_GrenadeToss.uasset
│   │   │   │   │   ├── AM_MM_Rifle_Melee.uasset
│   │   │   │   │   ├── AM_MM_Shotgun_Melee.uasset
│   │   │   │   │   ├── MF_Emote_FingerGuns.uasset
│   │   │   │   │   ├── MF_Emote_FingerGuns_Emote_MW.uasset
│   │   │   │   │   ├── MM_Dash_Backward.uasset
│   │   │   │   │   ├── MM_Dash_Forward.uasset
│   │   │   │   │   ├── MM_Dash_Left.uasset
│   │   │   │   │   ├── MM_Dash_Right.uasset
│   │   │   │   │   ├── MM_Death_Back_01.uasset
│   │   │   │   │   ├── MM_Death_Front_01.uasset
│   │   │   │   │   ├── MM_Death_Front_02.uasset
│   │   │   │   │   ├── MM_Death_Front_03.uasset
│   │   │   │   │   ├── MM_Death_Left_01.uasset
│   │   │   │   │   ├── MM_Death_Right_01.uasset
│   │   │   │   │   ├── MM_HitReact_Back_Lgt_01.uasset
│   │   │   │   │   ├── MM_HitReact_Back_Med_01.uasset
│   │   │   │   │   ├── MM_HitReact_Front_Hvy_01.uasset
│   │   │   │   │   ├── MM_HitReact_Front_Lgt_01.uasset
│   │   │   │   │   ├── MM_HitReact_Front_Lgt_02.uasset
│   │   │   │   │   ├── MM_HitReact_Front_Lgt_03.uasset
│   │   │   │   │   ├── MM_HitReact_Front_Lgt_04.uasset
│   │   │   │   │   ├── MM_HitReact_Front_Med_01.uasset
│   │   │   │   │   ├── MM_HitReact_Front_Med_02.uasset
│   │   │   │   │   ├── MM_HitReact_Left_Lgt_01.uasset
│   │   │   │   │   ├── MM_HitReact_Left_Med_01.uasset
│   │   │   │   │   ├── MM_HitReact_Right_Lgt_01.uasset
│   │   │   │   │   ├── MM_HitReact_Right_Med_01.uasset
│   │   │   │   │   ├── MM_Pistol_DryFire.uasset
│   │   │   │   │   ├── MM_Pistol_DryFire_Additive.uasset
│   │   │   │   │   ├── MM_Pistol_Equip.uasset
│   │   │   │   │   ├── MM_Pistol_Equip_Additive.uasset
│   │   │   │   │   ├── MM_Pistol_Fire.uasset
│   │   │   │   │   ├── MM_Pistol_Melee.uasset
│   │   │   │   │   ├── MM_Pistol_Melee_Additive.uasset
│   │   │   │   │   ├── MM_Pistol_Reload.uasset
│   │   │   │   │   ├── MM_Pistol_Reload_Additive.uasset
│   │   │   │   │   ├── MM_Pistol_Reload_Emote_MW.uasset
│   │   │   │   │   ├── MM_Pistol_Spawn.uasset
│   │   │   │   │   ├── MM_Pistol_Spawn_Slow.uasset
│   │   │   │   │   ├── MM_Pistol_Spawn_Turn180.uasset
│   │   │   │   │   ├── MM_Rifle_DryFire.uasset
│   │   │   │   │   ├── MM_Rifle_DryFire_Additive.uasset
│   │   │   │   │   ├── MM_Rifle_Equip.uasset
│   │   │   │   │   ├── MM_Rifle_Equip_Additive.uasset
│   │   │   │   │   ├── MM_Rifle_Fire.uasset
│   │   │   │   │   ├── MM_Rifle_GrenadeToss.uasset
│   │   │   │   │   ├── MM_Rifle_GrenadeToss_Additive.uasset
│   │   │   │   │   ├── MM_Rifle_Melee.uasset
│   │   │   │   │   ├── MM_Rifle_Melee_Additive.uasset
│   │   │   │   │   ├── MM_Rifle_Reload.uasset
│   │   │   │   │   ├── MM_Rifle_Reload_Additive.uasset
│   │   │   │   │   ├── MM_Rifle_Reload_Emote_MW.uasset
│   │   │   │   │   ├── MM_Rifle_Spawn.uasset
│   │   │   │   │   ├── MM_Rifle_Spawn_Fast.uasset
│   │   │   │   │   ├── MM_Rifle_Spawn_Turn180.uasset
│   │   │   │   │   ├── MM_Shotgun_Fire.uasset
│   │   │   │   │   ├── MM_Shotgun_Melee.uasset
│   │   │   │   │   ├── MM_Shotgun_Melee_Additive.uasset
│   │   │   │   │   ├── MM_Shotgun_Reload.uasset
│   │   │   │   │   ├── MM_Shotgun_Reload_Additive.uasset
│   │   │   │   │   └── MM_Shotgun_Reload_Emote_MW.uasset
│   │   │   │   ├── AimOffsets/
│   │   │   │   │   ├── AO_MF_Pistol_Idle_ADS.uasset
│   │   │   │   │   ├── AO_MM_Pistol_Idle_ADS.uasset
│   │   │   │   │   ├── AO_MM_Rifle_Crouch_Idle.uasset
│   │   │   │   │   ├── AO_MM_Rifle_Idle_ADS.uasset
│   │   │   │   │   ├── AO_MM_Rifle_Idle_Hipfire.uasset
│   │   │   │   │   ├── AO_MM_Unarmed_Idle_Ready.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_CC.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_CD.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_CU.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_LBC.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_LBD.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_LBU.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_LC.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_LD.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_LU.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_RBC.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_RBD.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_RBU.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_RC.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_RD.uasset
│   │   │   │   │   ├── MF_Pistol_Idle_ADS_AO_RU.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_CC.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_CD.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_CD_45.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_CU.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_CU_45.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_LBC.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_LBD.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_LBU.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_LC.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_LD.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_LU.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_RBC.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_RBD.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_RBU.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_RC.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_RD.uasset
│   │   │   │   │   ├── MM_Pistol_Idle_ADS_AO_RU.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_CC.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_CD.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_CU.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_LBC.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_LBD.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_LBU.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_LC.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_LD.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_LU.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_RBC.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_RBD.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_RBU.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_RC.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_RD.uasset
│   │   │   │   │   ├── MM_Rifle_Crouch_Idle_AO_RU.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_CC.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_CD.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_CU.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_LBC.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_LBD.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_LBU.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_LC.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_LD.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_LU.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_RBC.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_RBD.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_RBU.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_RC.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_RD.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_ADS_AO_RU.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_CC.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_CD.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_CU.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_LBC.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_LBD.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_LBU.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_LC.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_LD.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_LU.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_RBC.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_RBD.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_RBU.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_RC.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_RD.uasset
│   │   │   │   │   ├── MM_Rifle_Idle_Hipfire_AO_RU.uasset
│   │   │   │   │   ├── MM_Unarmed_Idle_Ready_AO_CC.uasset
│   │   │   │   │   ├── MM_Unarmed_Idle_Ready_AO_CD.uasset
│   │   │   │   │   ├── MM_Unarmed_Idle_Ready_AO_CU.uasset
│   │   │   │   │   ├── MM_Unarmed_Idle_Ready_AO_LBC.uasset
│   │   │   │   │   ├── MM_Unarmed_Idle_Ready_AO_LBD.uasset
│   │   │   │   │   ├── MM_Unarmed_Idle_Ready_AO_LBU.uasset
│   │   │   │   │   ├── MM_Unarmed_Idle_Ready_AO_LC.uasset
│   │   │   │   │   ├── MM_Unarmed_Idle_Ready_AO_LD.uasset
│   │   │   │   │   ├── MM_Unarmed_Idle_Ready_AO_LU.uasset
│   │   │   │   │   ├── MM_Unarmed_Idle_Ready_AO_RBC.uasset
│   │   │   │   │   ├── MM_Unarmed_Idle_Ready_AO_RBD.uasset
│   │   │   │   │   ├── MM_Unarmed_Idle_Ready_AO_RBU.uasset
│   │   │   │   │   ├── MM_Unarmed_Idle_Ready_AO_RC.uasset
│   │   │   │   │   ├── MM_Unarmed_Idle_Ready_AO_RD.uasset
│   │   │   │   │   └── MM_Unarmed_Idle_Ready_AO_RU.uasset
│   │   │   │   ├── AnimModifiers/
│   │   │   │   │   ├── FootFXAnimModifier.uasset
│   │   │   │   │   ├── FootFXAnimModifier_FootDefinition.uasset
│   │   │   │   │   ├── FootstepEffectTagModifier.uasset
│   │   │   │   │   └── TurnYawAnimModifier.uasset
│   │   │   │   ├── AnimNotifies/
│   │   │   │   │   ├── AN_PlayWeaponMontage.uasset
│   │   │   │   │   └── TransitionToLocomotion.uasset
│   │   │   │   ├── Interactions/
│   │   │   │   │   └── Bench/
│   │   │   │   │       ├── int_sit_bench_sit_convo_L.uasset
│   │   │   │   │       ├── int_sit_bench_sit_convo_L_Montage.uasset
│   │   │   │   │       ├── int_sit_bench_sit_convo_R.uasset
│   │   │   │   │       ├── int_sit_bench_sit_convo_R_Montage.uasset
│   │   │   │   │       ├── int_sit_bench_sit_idle_loop.uasset
│   │   │   │   │       ├── int_sit_bench_sit_idle_loop_Montage.uasset
│   │   │   │   │       ├── int_sit_bench_sit_idle_twitch_v1.uasset
│   │   │   │   │       ├── int_sit_bench_sit_idle_twitch_v1_Montage.uasset
│   │   │   │   │       ├── int_sit_bench_sit_idle_twitch_v2.uasset
│   │   │   │   │       ├── int_sit_bench_sit_idle_twitch_v2_Montage.uasset
│   │   │   │   │       ├── int_sit_bench_sit_idle_twitch_v3.uasset
│   │   │   │   │       ├── int_sit_bench_sit_idle_twitch_v3_Montage.uasset
│   │   │   │   │       ├── int_sit_bench_sit_into_L_45.uasset
│   │   │   │   │       ├── int_sit_bench_sit_into_L_45_Montage.uasset
│   │   │   │   │       ├── int_sit_bench_sit_into_L_90.uasset
│   │   │   │   │       ├── int_sit_bench_sit_into_L_90_Montage.uasset
│   │   │   │   │       ├── int_sit_bench_sit_into_R_45.uasset
│   │   │   │   │       ├── int_sit_bench_sit_into_R_45_Montage.uasset
│   │   │   │   │       ├── int_sit_bench_sit_into_R_90.uasset
│   │   │   │   │       ├── int_sit_bench_sit_into_R_90_Montage.uasset
│   │   │   │   │       ├── int_sit_bench_sit_into_fwd.uasset
│   │   │   │   │       ├── int_sit_bench_sit_into_fwd_Montage.uasset
│   │   │   │   │       ├── int_sit_bench_sit_out_run.uasset
│   │   │   │   │       ├── int_sit_bench_sit_out_run_Montage.uasset
│   │   │   │   │       ├── int_sit_bench_sit_out_stand.uasset
│   │   │   │   │       ├── int_sit_bench_sit_out_stand_Montage.uasset
│   │   │   │   │       ├── int_sit_bench_sit_out_walk.uasset
│   │   │   │   │       └── int_sit_bench_sit_out_walk_Montage.uasset
│   │   │   │   ├── LinkedLayers/
│   │   │   │   │   ├── ABP_ItemAnimLayersBase.uasset
│   │   │   │   │   └── ALI_ItemAnimLayers.uasset
│   │   │   │   ├── Locomotion/
│   │   │   │   │   ├── Pistol/
│   │   │   │   │   │   ├── ABP_PistolAnimLayers.uasset
│   │   │   │   │   │   ├── ABP_PistolAnimLayers_Feminine.uasset
│   │   │   │   │   │   ├── MF_Pistol_Hipfire_OverridePose.uasset
│   │   │   │   │   │   ├── MF_Pistol_IdleBreak_Scan.uasset
│   │   │   │   │   │   ├── MF_Pistol_Idle_ADS.uasset
│   │   │   │   │   │   ├── MF_Pistol_Idle_Hipfire.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Bwd.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Bwd_Pivot.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Bwd_Start.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Bwd_Stop.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Fwd.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Fwd_Pivot.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Fwd_Start.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Fwd_Stop.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Left.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Left_Pivot.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Left_Start.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Left_Stop.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Right.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Right_Pivot.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Right_Start.uasset
│   │   │   │   │   │   ├── MF_Pistol_Jog_Right_Stop.uasset
│   │   │   │   │   │   ├── MF_Pistol_TurnLeft_180.uasset
│   │   │   │   │   │   ├── MF_Pistol_TurnLeft_90.uasset
│   │   │   │   │   │   ├── MF_Pistol_TurnRight_180.uasset
│   │   │   │   │   │   ├── MF_Pistol_TurnRight_90.uasset
│   │   │   │   │   │   ├── MF_Pistol_Walk_Bwd.uasset
│   │   │   │   │   │   ├── MF_Pistol_Walk_Bwd_Start.uasset
│   │   │   │   │   │   ├── MF_Pistol_Walk_Bwd_Stop.uasset
│   │   │   │   │   │   ├── MF_Pistol_Walk_Fwd.uasset
│   │   │   │   │   │   ├── MF_Pistol_Walk_Fwd_Start.uasset
│   │   │   │   │   │   ├── MF_Pistol_Walk_Fwd_Stop.uasset
│   │   │   │   │   │   ├── MF_Pistol_Walk_Left.uasset
│   │   │   │   │   │   ├── MF_Pistol_Walk_Left_Stop.uasset
│   │   │   │   │   │   ├── MF_Pistol_Walk_Right.uasset
│   │   │   │   │   │   ├── MF_Pistol_Walk_Right_Start.uasset
│   │   │   │   │   │   ├── MF_Pistol_Walk_Right_Stop.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Entry.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Exit.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Idle.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_OverridePose.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_TurnLeft_180.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_TurnLeft_90.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_TurnRight_180.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_TurnRight_90.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Bwd.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Bwd_Pivot.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Bwd_Start.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Bwd_Stop.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Fwd.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Fwd_Pivot.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Fwd_Start.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Fwd_Stop.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Left.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Left_Pivot.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Left_Start.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Left_Stop.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Right.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Right_Pivot.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Right_Start.uasset
│   │   │   │   │   │   ├── MM_Pistol_Crouch_Walk_Right_Stop.uasset
│   │   │   │   │   │   ├── MM_Pistol_IdleBreak_Scan.uasset
│   │   │   │   │   │   ├── MM_Pistol_Idle_ADS.uasset
│   │   │   │   │   │   ├── MM_Pistol_Idle_Hipfire.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Bwd.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Bwd_Start.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Bwd_Stop.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Fwd.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Fwd_Start.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Fwd_Stop.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Left.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Left_Start.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Left_Stop.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Pivot_Bwd.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Pivot_Fwd.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Pivot_Left.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Pivot_Right.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Right.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Right_Start.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Right_Stop.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jog_Stop.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jump_Apex.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jump_Fall_Land.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jump_Fall_Loop.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jump_RecoveryAdditive.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jump_Start.uasset
│   │   │   │   │   │   ├── MM_Pistol_Jump_Start_Loop.uasset
│   │   │   │   │   │   ├── MM_Pistol_TurnLeft_180.uasset
│   │   │   │   │   │   ├── MM_Pistol_TurnLeft_90.uasset
│   │   │   │   │   │   ├── MM_Pistol_TurnRight_180.uasset
│   │   │   │   │   │   ├── MM_Pistol_TurnRight_90.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Bwd.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Bwd_Pivot.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Bwd_Start.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Bwd_Stop.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Fwd.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Fwd_Pivot.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Fwd_Start.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Fwd_Stop.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Left.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Left_Pivot.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Left_Start.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Left_Stop.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Right.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Right_Pivot.uasset
│   │   │   │   │   │   ├── MM_Pistol_Walk_Right_Start.uasset
│   │   │   │   │   │   └── MM_Pistol_Walk_Right_Stop.uasset
│   │   │   │   │   ├── Rifle/
│   │   │   │   │   │   ├── ABP_RifleAnimLayers.uasset
│   │   │   │   │   │   ├── ABP_RifleAnimLayers_Feminine.uasset
│   │   │   │   │   │   ├── BS_MM_Rifle_Crouch_Walk.uasset
│   │   │   │   │   │   ├── BS_MM_Rifle_Jog_Leans.uasset
│   │   │   │   │   │   ├── MF_Rifle_Idle_ADS.uasset
│   │   │   │   │   │   ├── MF_Rifle_Idle_Hipfire.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Bwd.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Bwd_Pivot.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Bwd_Start.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Bwd_Stop.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Fwd.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Fwd_Pivot.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Fwd_Start.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Fwd_Stop.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Left.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Left_Pivot.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Left_Start.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Left_Stop.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Right.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Right_Pivot.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Right_Start.uasset
│   │   │   │   │   │   ├── MF_Rifle_Jog_Right_Stop.uasset
│   │   │   │   │   │   ├── MF_Rifle_TurnLeft_180.uasset
│   │   │   │   │   │   ├── MF_Rifle_TurnLeft_90.uasset
│   │   │   │   │   │   ├── MF_Rifle_TurnRight_180.uasset
│   │   │   │   │   │   ├── MF_Rifle_TurnRight_90.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Bwd.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Bwd_Pivot.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Bwd_Start.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Bwd_Stop.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Fwd.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Fwd_Pivot.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Fwd_Start.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Fwd_Stop.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Left.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Left_Pivot.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Left_Start.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Left_Stop.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Right.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Right_Pivot.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Right_Start.uasset
│   │   │   │   │   │   ├── MF_Rifle_Walk_Right_Stop.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Entry.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Exit.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Idle.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_OverridePose.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_TurnLeft_180.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_TurnLeft_90.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_TurnRight_180.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_TurnRight_90.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Bwd.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Bwd_Pivot.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Bwd_Start.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Bwd_Stop.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Fwd.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Fwd_Pivot.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Fwd_Start.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Fwd_Stop.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Left.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Left_Pivot.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Left_Start.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Left_Stop.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Right.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Right_Pivot.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Right_Start.uasset
│   │   │   │   │   │   ├── MM_Rifle_Crouch_Walk_Right_Stop.uasset
│   │   │   │   │   │   ├── MM_Rifle_Hipfire_OverridePose.uasset
│   │   │   │   │   │   ├── MM_Rifle_IdleBreak_Fidget.uasset
│   │   │   │   │   │   ├── MM_Rifle_IdleBreak_Scan.uasset
│   │   │   │   │   │   ├── MM_Rifle_Idle_ADS.uasset
│   │   │   │   │   │   ├── MM_Rifle_Idle_Hipfire.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Bwd.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Bwd_Pivot.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Bwd_Start.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Bwd_Stop.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Fwd.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Fwd_Pivot.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Fwd_RAW.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Fwd_Start.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Fwd_Stop.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Lean_Center.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Lean_Right.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Leans_Left.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Left.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Left_Pivot.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Left_Start.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Left_Stop.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Right.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Right_Pivot.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Right_Start.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jog_Right_Stop.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jump_Apex.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jump_Fall_Land.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jump_Fall_Loop.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jump_RecoveryAdditive.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jump_Start.uasset
│   │   │   │   │   │   ├── MM_Rifle_Jump_Start_Loop.uasset
│   │   │   │   │   │   ├── MM_Rifle_TurnLeft_180.uasset
│   │   │   │   │   │   ├── MM_Rifle_TurnLeft_90.uasset
│   │   │   │   │   │   ├── MM_Rifle_TurnRight_180.uasset
│   │   │   │   │   │   ├── MM_Rifle_TurnRight_90.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Bwd.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Bwd_Pivot.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Bwd_Start.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Bwd_Stop.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Fwd.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Fwd_Pivot.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Fwd_Start.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Fwd_Stop.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Left.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Left_Pivot.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Left_Start.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Left_Stop.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Right.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Right_Pivot.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Right_Start.uasset
│   │   │   │   │   │   ├── MM_Rifle_Walk_Right_Stop.uasset
│   │   │   │   │   │   └── Mf_Rifle_IdleBreak_Fidget.uasset
│   │   │   │   │   ├── Shotgun/
│   │   │   │   │   │   ├── ABP_ShotgunAnimLayers.uasset
│   │   │   │   │   │   ├── ABP_ShotgunAnimLayers_Feminine.uasset
│   │   │   │   │   │   ├── MF_Shotgun_Idle_Hipfire.uasset
│   │   │   │   │   │   ├── MM_Shotgun_Idle_ADS.uasset
│   │   │   │   │   │   └── MM_Shotgun_Idle_Hipfire.uasset
│   │   │   │   │   └── Unarmed/
│   │   │   │   │       ├── ABP_UnarmedAnimLayers.uasset
│   │   │   │   │       ├── ABP_UnarmedAnimLayers_Feminine.uasset
│   │   │   │   │       ├── BS_MM_Unarmed_Jog_Walk.uasset
│   │   │   │   │       ├── MF_Unarmed_Idle_Break.uasset
│   │   │   │   │       ├── MF_Unarmed_Idle_Ready.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Bwd.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Bwd_Pivot.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Bwd_Start.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Bwd_Stop.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Fwd.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Fwd_Pivot.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Fwd_Start.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Fwd_Stop.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Left.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Left_Pivot.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Left_Start.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Left_Stop.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Right.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Right_Pivot.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Right_Start.uasset
│   │   │   │   │       ├── MF_Unarmed_Jog_Right_Stop.uasset
│   │   │   │   │       ├── MF_Unarmed_TurnLeft_180.uasset
│   │   │   │   │       ├── MF_Unarmed_TurnLeft_90.uasset
│   │   │   │   │       ├── MF_Unarmed_TurnRight_180.uasset
│   │   │   │   │       ├── MF_Unarmed_TurnRight_90.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Bwd.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Bwd_Pivot.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Bwd_Start.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Bwd_Stop.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Fwd.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Fwd_Pivot.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Fwd_Start.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Fwd_Stop.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Left.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Left_Pivot.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Left_Start.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Left_Stop.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Right.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Right_Pivot.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Right_Start.uasset
│   │   │   │   │       ├── MF_Unarmed_Walk_Right_Stop.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Entry.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Exit.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Idle.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_TurnLeft_180.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_TurnLeft_90.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_TurnRight_180.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_TurnRight_90.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Bwd.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Bwd_Pivot.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Bwd_Start.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Bwd_Stop.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Fwd.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Fwd_Pivot.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Fwd_Start.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Fwd_Stop.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Left.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Left_Pivot.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Left_Start.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Left_Stop.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Right.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Right_Pivot.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Right_Start.uasset
│   │   │   │   │       ├── MM_Unarmed_Crouch_Walk_Right_Stop.uasset
│   │   │   │   │       ├── MM_Unarmed_IdleBreak_Fidget.uasset
│   │   │   │   │       ├── MM_Unarmed_IdleBreak_Scan.uasset
│   │   │   │   │       ├── MM_Unarmed_Idle_Ready.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Bwd.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Bwd_Pivot.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Bwd_Start.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Bwd_Stop.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Fwd.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Fwd_Pivot.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Fwd_Start.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Fwd_Stop.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Left.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Left_Pivot.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Left_Start.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Left_Stop.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Pivot_Right.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Right.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Right_Start.uasset
│   │   │   │   │       ├── MM_Unarmed_Jog_Right_Stop.uasset
│   │   │   │   │       ├── MM_Unarmed_Jump_Apex.uasset
│   │   │   │   │       ├── MM_Unarmed_Jump_Fall_Land.uasset
│   │   │   │   │       ├── MM_Unarmed_Jump_Fall_Loop.uasset
│   │   │   │   │       ├── MM_Unarmed_Jump_RecoveryAdditive.uasset
│   │   │   │   │       ├── MM_Unarmed_Jump_Start.uasset
│   │   │   │   │       ├── MM_Unarmed_Jump_Start_Loop.uasset
│   │   │   │   │       ├── MM_Unarmed_TurnLeft_180.uasset
│   │   │   │   │       ├── MM_Unarmed_TurnLeft_90.uasset
│   │   │   │   │       ├── MM_Unarmed_TurnRight_180.uasset
│   │   │   │   │       ├── MM_Unarmed_TurnRight_90.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Bwd.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Bwd_Pivot.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Bwd_Start.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Bwd_Stop.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Fwd.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Fwd_Pivot.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Fwd_Start.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Fwd_Stop.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Left.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Left_Pivot.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Left_Start.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Left_Stop.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Right.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Right_Pivot.uasset
│   │   │   │   │       ├── MM_Unarmed_Walk_Right_Start.uasset
│   │   │   │   │       └── MM_Unarmed_Walk_Right_Stop.uasset
│   │   │   │   ├── Poses/
│   │   │   │   │   ├── QuinnIntro_Blockout/
│   │   │   │   │   │   ├── QuinnIntro_BlockOut_Pose1_Manny.uasset
│   │   │   │   │   │   ├── QuinnIntro_BlockOut_Pose1_Quinn.uasset
│   │   │   │   │   │   ├── QuinnIntro_BlockOut_Pose2_Manny.uasset
│   │   │   │   │   │   ├── QuinnIntro_BlockOut_Pose2_Quinn.uasset
│   │   │   │   │   │   ├── QuinnIntro_BlockOut_Pose3_Manny.uasset
│   │   │   │   │   │   ├── QuinnIntro_BlockOut_Pose3_Quinn.uasset
│   │   │   │   │   │   ├── QuinnIntro_BlockOut_Pose4_Manny.uasset
│   │   │   │   │   │   ├── QuinnIntro_BlockOut_Pose4_Quinn.uasset
│   │   │   │   │   │   ├── QuinnIntro_BlockOut_Pose5_Manny.uasset
│   │   │   │   │   │   ├── QuinnIntro_BlockOut_Pose5_Quinn.uasset
│   │   │   │   │   │   ├── QuinnIntro_BlockOut_Pose6_Manny.uasset
│   │   │   │   │   │   ├── QuinnIntro_BlockOut_Pose6_Quinn.uasset
│   │   │   │   │   │   ├── QuinnIntro_BlockOut_Pose7_Manny.uasset
│   │   │   │   │   │   └── QuinnIntro_BlockOut_Pose7_Quinn.uasset
│   │   │   │   │   ├── MM_Dash_Forward_LoadingScreenStills.uasset
│   │   │   │   │   ├── MM_Dash_Forward_LoadingScreen_Still_A.uasset
│   │   │   │   │   ├── MM_Dash_Forward_LoadingScreen_Still_B.uasset
│   │   │   │   │   ├── MM_Dash_Forward_LoadingScreen_Still_C.uasset
│   │   │   │   │   ├── MM_Dash_Forward_LoadingScreen_Still_D.uasset
│   │   │   │   │   ├── MM_Dash_Forward_LoadingScreen_Still_E.uasset
│   │   │   │   │   ├── SplashPose_1.uasset
│   │   │   │   │   ├── SplashPose_10.uasset
│   │   │   │   │   ├── SplashPose_11.uasset
│   │   │   │   │   ├── SplashPose_12.uasset
│   │   │   │   │   ├── SplashPose_13.uasset
│   │   │   │   │   ├── SplashPose_14.uasset
│   │   │   │   │   ├── SplashPose_15.uasset
│   │   │   │   │   ├── SplashPose_16.uasset
│   │   │   │   │   ├── SplashPose_2.uasset
│   │   │   │   │   ├── SplashPose_3.uasset
│   │   │   │   │   ├── SplashPose_4.uasset
│   │   │   │   │   ├── SplashPose_5.uasset
│   │   │   │   │   ├── SplashPose_6.uasset
│   │   │   │   │   ├── SplashPose_7.uasset
│   │   │   │   │   ├── SplashPose_8.uasset
│   │   │   │   │   ├── SplashPose_9.uasset
│   │   │   │   │   ├── SplashPose_Quinn_1.uasset
│   │   │   │   │   ├── SplashPose_Quinn_10.uasset
│   │   │   │   │   ├── SplashPose_Quinn_11.uasset
│   │   │   │   │   ├── SplashPose_Quinn_12.uasset
│   │   │   │   │   ├── SplashPose_Quinn_13.uasset
│   │   │   │   │   ├── SplashPose_Quinn_14.uasset
│   │   │   │   │   ├── SplashPose_Quinn_15.uasset
│   │   │   │   │   ├── SplashPose_Quinn_16.uasset
│   │   │   │   │   ├── SplashPose_Quinn_17.uasset
│   │   │   │   │   ├── SplashPose_Quinn_18.uasset
│   │   │   │   │   ├── SplashPose_Quinn_2.uasset
│   │   │   │   │   ├── SplashPose_Quinn_3.uasset
│   │   │   │   │   ├── SplashPose_Quinn_4.uasset
│   │   │   │   │   ├── SplashPose_Quinn_5.uasset
│   │   │   │   │   ├── SplashPose_Quinn_6.uasset
│   │   │   │   │   ├── SplashPose_Quinn_7.uasset
│   │   │   │   │   ├── SplashPose_Quinn_8.uasset
│   │   │   │   │   ├── SplashPose_Quinn_9.uasset
│   │   │   │   │   └── SplashPose_SmearPoses.uasset
│   │   │   │   ├── ABP_Mannequin_Base.uasset
│   │   │   │   ├── ABP_Mannequin_CopyPose.uasset
│   │   │   │   ├── ABP_Mannequin_Retarget.uasset
│   │   │   │   ├── ABP_Mannequin_TopDown.uasset
│   │   │   │   ├── AnimEnum_CardinalDirection.uasset
│   │   │   │   ├── AnimEnum_RootYawOffsetMode.uasset
│   │   │   │   ├── AnimStruct_CardinalDirections.uasset
│   │   │   │   ├── AnimStruct_TurnInPlaceEntry.uasset
│   │   │   │   └── UniformIndexableCurveCompressionSettings.uasset
│   │   │   ├── Materials/
│   │   │   │   ├── Functions/
│   │   │   │   │   ├── CA_Mannequin_new.uasset
│   │   │   │   │   ├── ChromaticCurve2.uasset
│   │   │   │   │   ├── MF_BaseColorFallOff.uasset
│   │   │   │   │   ├── MF_Diffraction.uasset
│   │   │   │   │   └── MF_logo.uasset
│   │   │   │   ├── Instances/
│   │   │   │   │   ├── Manny/
│   │   │   │   │   │   ├── MI_Manny_01.uasset
│   │   │   │   │   │   ├── MI_Manny_01_Blue.uasset
│   │   │   │   │   │   ├── MI_Manny_01_Red.uasset
│   │   │   │   │   │   ├── MI_Manny_01_SMEAR.uasset
│   │   │   │   │   │   ├── MI_Manny_02.uasset
│   │   │   │   │   │   ├── MI_Manny_02_Blue.uasset
│   │   │   │   │   │   ├── MI_Manny_02_Red.uasset
│   │   │   │   │   │   └── MI_Manny_02_SMEAR.uasset
│   │   │   │   │   └── Quinn/
│   │   │   │   │       ├── MI_Quinn_01.uasset
│   │   │   │   │       ├── MI_Quinn_01_Blue.uasset
│   │   │   │   │       ├── MI_Quinn_01_Red.uasset
│   │   │   │   │       ├── MI_Quinn_02.uasset
│   │   │   │   │       ├── MI_Quinn_02_Blue.uasset
│   │   │   │   │       └── MI_Quinn_02_Red.uasset
│   │   │   │   ├── M_Mannequin.uasset
│   │   │   │   └── M_MannequinSmear.uasset
│   │   │   ├── Meshes/
│   │   │   │   ├── Mannequin_LODSettings.uasset
│   │   │   │   ├── SKM_Manny.uasset
│   │   │   │   ├── SKM_Manny_Invis.uasset
│   │   │   │   ├── SKM_Quinn.uasset
│   │   │   │   ├── SKM_Quinn_Invis.uasset
│   │   │   │   └── SK_Mannequin.uasset
│   │   │   ├── Rig/
│   │   │   │   ├── Poses/
│   │   │   │   │   ├── Manny/
│   │   │   │   │   │   ├── Manny_calf_l_anim.uasset
│   │   │   │   │   │   ├── Manny_calf_l_pose.uasset
│   │   │   │   │   │   ├── Manny_calf_r_anim.uasset
│   │   │   │   │   │   ├── Manny_calf_r_pose.uasset
│   │   │   │   │   │   ├── Manny_clavicle_l_anim.uasset
│   │   │   │   │   │   ├── Manny_clavicle_l_pose.uasset
│   │   │   │   │   │   ├── Manny_clavicle_r_anim.uasset
│   │   │   │   │   │   ├── Manny_clavicle_r_pose.uasset
│   │   │   │   │   │   ├── Manny_foot_l_anim.uasset
│   │   │   │   │   │   ├── Manny_foot_l_pose.uasset
│   │   │   │   │   │   ├── Manny_foot_r_anim.uasset
│   │   │   │   │   │   ├── Manny_foot_r_pose.uasset
│   │   │   │   │   │   ├── Manny_hand_l_anim.uasset
│   │   │   │   │   │   ├── Manny_hand_l_pose.uasset
│   │   │   │   │   │   ├── Manny_hand_r_anim.uasset
│   │   │   │   │   │   ├── Manny_hand_r_pose.uasset
│   │   │   │   │   │   ├── Manny_lowerarm_l_anim.uasset
│   │   │   │   │   │   ├── Manny_lowerarm_l_pose.uasset
│   │   │   │   │   │   ├── Manny_lowerarm_r_anim.uasset
│   │   │   │   │   │   ├── Manny_lowerarm_r_pose.uasset
│   │   │   │   │   │   ├── Manny_thigh_l_anim.uasset
│   │   │   │   │   │   ├── Manny_thigh_l_pose.uasset
│   │   │   │   │   │   ├── Manny_thigh_r_anim.uasset
│   │   │   │   │   │   ├── Manny_thigh_r_pose.uasset
│   │   │   │   │   │   ├── Manny_upperarm_l_anim.uasset
│   │   │   │   │   │   ├── Manny_upperarm_l_pose.uasset
│   │   │   │   │   │   ├── Manny_upperarm_r_anim.uasset
│   │   │   │   │   │   └── Manny_upperarm_r_pose.uasset
│   │   │   │   │   └── Quinn/
│   │   │   │   │       ├── Quinn_calf_l_anim.uasset
│   │   │   │   │       ├── Quinn_calf_l_pose.uasset
│   │   │   │   │       ├── Quinn_calf_r_anim.uasset
│   │   │   │   │       ├── Quinn_calf_r_pose.uasset
│   │   │   │   │       ├── Quinn_clavicle_l_anim.uasset
│   │   │   │   │       ├── Quinn_clavicle_l_pose.uasset
│   │   │   │   │       ├── Quinn_clavicle_r_anim.uasset
│   │   │   │   │       ├── Quinn_clavicle_r_pose.uasset
│   │   │   │   │       ├── Quinn_foot_l_anim.uasset
│   │   │   │   │       ├── Quinn_foot_l_pose.uasset
│   │   │   │   │       ├── Quinn_foot_r_anim.uasset
│   │   │   │   │       ├── Quinn_foot_r_pose.uasset
│   │   │   │   │       ├── Quinn_hand_l_anim.uasset
│   │   │   │   │       ├── Quinn_hand_l_pose.uasset
│   │   │   │   │       ├── Quinn_hand_r_anim.uasset
│   │   │   │   │       ├── Quinn_hand_r_pose.uasset
│   │   │   │   │       ├── Quinn_lowerarm_l_anim.uasset
│   │   │   │   │       ├── Quinn_lowerarm_l_pose.uasset
│   │   │   │   │       ├── Quinn_lowerarm_r_anim.uasset
│   │   │   │   │       ├── Quinn_lowerarm_r_pose.uasset
│   │   │   │   │       ├── Quinn_thigh_l_anim.uasset
│   │   │   │   │       ├── Quinn_thigh_l_pose.uasset
│   │   │   │   │       ├── Quinn_thigh_r_anim.uasset
│   │   │   │   │       ├── Quinn_thigh_r_pose.uasset
│   │   │   │   │       ├── Quinn_upperarm_l_anim.uasset
│   │   │   │   │       ├── Quinn_upperarm_l_pose.uasset
│   │   │   │   │       ├── Quinn_upperarm_r_anim.uasset
│   │   │   │   │       └── Quinn_upperarm_r_pose.uasset
│   │   │   │   ├── ABP_Manny_PostProcess.uasset
│   │   │   │   ├── ABP_Quinn_PostProcess.uasset
│   │   │   │   ├── CR_Mannequin_Body.uasset
│   │   │   │   ├── CR_Mannequin_FootPlant.uasset
│   │   │   │   ├── CR_Mannequin_Procedural.uasset
│   │   │   │   ├── IK_Mannequin.uasset
│   │   │   │   ├── PA_Mannequin.uasset
│   │   │   │   └── RTG_Mannequin.uasset
│   │   │   └── Textures/
│   │   │       ├── Manny/
│   │   │       │   ├── T_Manny_01_D.uasset
│   │   │       │   ├── T_Manny_01_MSK.uasset
│   │   │       │   ├── T_Manny_01_MSK1.uasset
│   │   │       │   ├── T_Manny_01_MSK2.uasset
│   │   │       │   ├── T_Manny_01_MSK3.uasset
│   │   │       │   ├── T_Manny_01_N.uasset
│   │   │       │   ├── T_Manny_02_D.uasset
│   │   │       │   ├── T_Manny_02_MSK.uasset
│   │   │       │   ├── T_Manny_02_MSK1.uasset
│   │   │       │   ├── T_Manny_02_MSK2.uasset
│   │   │       │   ├── T_Manny_02_MSK3.uasset
│   │   │       │   └── T_Manny_02_N.uasset
│   │   │       ├── Quinn/
│   │   │       │   ├── T_Quinn_01_D.uasset
│   │   │       │   ├── T_Quinn_01_MSK.uasset
│   │   │       │   ├── T_Quinn_01_MSK1.uasset
│   │   │       │   ├── T_Quinn_01_MSK2.uasset
│   │   │       │   ├── T_Quinn_01_MSK3.uasset
│   │   │       │   ├── T_Quinn_01_N.uasset
│   │   │       │   ├── T_Quinn_02_D.uasset
│   │   │       │   ├── T_Quinn_02_MSK.uasset
│   │   │       │   ├── T_Quinn_02_MSK1.uasset
│   │   │       │   ├── T_Quinn_02_MSK2.uasset
│   │   │       │   ├── T_Quinn_02_MSK3.uasset
│   │   │       │   └── T_Quinn_02_N.uasset
│   │   │       └── Shared/
│   │   │           ├── CA_Mannequin.uasset
│   │   │           ├── ChromaticCurve.uasset
│   │   │           ├── T_Detail_Normal.uasset
│   │   │           ├── T_OrangePeel_N.uasset
│   │   │           └── T_UE_Logo_V2.uasset
│   │   ├── Mannequin_UE4/
│   │   │   ├── Animations/
│   │   │   │   └── ABP_UE4_Mannequin_Retarget.uasset
│   │   │   ├── Materials/
│   │   │   │   ├── Layers/
│   │   │   │   │   ├── ML_Latex_Black.uasset
│   │   │   │   │   ├── ML_ShinyPlastic_Beige.uasset
│   │   │   │   │   ├── ML_ShinyPlastic_Beige_Logo.uasset
│   │   │   │   │   └── ML_SoftMetal.uasset
│   │   │   │   ├── Textures/
│   │   │   │   │   ├── T_ML_Aluminum01.uasset
│   │   │   │   │   ├── T_ML_Aluminum01_N.uasset
│   │   │   │   │   ├── T_ML_Rubber_Blue_01_D.uasset
│   │   │   │   │   ├── T_ML_Rubber_Blue_01_N.uasset
│   │   │   │   │   ├── UE4Man_Logo_N.uasset
│   │   │   │   │   ├── UE4_LOGO_CARD.uasset
│   │   │   │   │   ├── UE4_Mannequin_MAT_MASKA.uasset
│   │   │   │   │   └── UE4_Mannequin__normals.uasset
│   │   │   │   ├── M_MannequinUE4_Body.uasset
│   │   │   │   └── M_MannequinUE4_ChestLogo.uasset
│   │   │   └── Meshes/
│   │   │       ├── IK_UE4_Mannequin.uasset
│   │   │       ├── RTG_UE4Manny_UE5Manny.uasset
│   │   │       ├── RTG_UE5Manny_UE4Manny.uasset
│   │   │       ├── SK_Mannequin.uasset
│   │   │       ├── SK_Mannequin_PhysicsAsset.uasset
│   │   │       └── SK_Mannequin_Skeleton.uasset
│   │   ├── SimplePawnData/
│   │   │   └── SimplePawnData.uasset
│   │   ├── B_Hero_Default.uasset
│   │   ├── B_SimpleHeroPawn.uasset
│   │   ├── M_SimpleHeroPawn.uasset
│   │   ├── PhysMat_Player.uasset
│   │   └── PhysMat_Player_WeakSpot.uasset
│   └── Character_Default.uasset
├── ContextEffects/
│   ├── CFX_DefaultSkin.uasset
│   ├── DT_AnimEffectTags.uasset
│   └── DT_SurfaceTypes.uasset
├── Editor/
│   └── Slate/
│       └── Icons/
├── Effects/
│   ├── AnimationNotifies/
│   │   ├── AN_FootPlant_Left.uasset
│   │   └── AN_FootPlant_Right.uasset
│   ├── Blueprints/
│   │   ├── B_FootStep.uasset
│   │   ├── B_NiagaraNumberPopComponent.uasset
│   │   ├── B_WeaponDecals.uasset
│   │   ├── B_WeaponFire.uasset
│   │   ├── B_WeaponImpacts.uasset
│   │   ├── Damage_BasicNiagaraStyle.uasset
│   │   └── SurfaceImpacts.uasset
│   ├── Camera/
│   │   └── Damage/
│   │       ├── Materials/
│   │       │   ├── Instances/
│   │       │   │   ├── MI_ScreenDamageParticles.uasset
│   │       │   │   └── MI_ScreenDamageParticlesBlurry.uasset
│   │       │   ├── M_DamageDirection.uasset
│   │       │   ├── M_ScreenDamageParticles.uasset
│   │       │   └── M_ScreenDamageParticlesBlurry.uasset
│   │       ├── Textures/
│   │       │   └── Hit_Regular_Glitch.uasset
│   │       ├── NCLE_DamageTaken.uasset
│   │       └── NS_Screen_DamageDirection.uasset
│   ├── Curves/
│   │   ├── Curve_LaunchpadMaterialEffect.uasset
│   │   ├── FX_Pyro.uasset
│   │   ├── FX_Pyro_Fire.uasset
│   │   ├── FX_Pyro_Fire_2.uasset
│   │   ├── FX_Pyro_Fire_3.uasset
│   │   └── FX_Pyro_Sparks.uasset
│   ├── DataChannels/
│   │   └── ImpactDataChannel.uasset
│   ├── EffectTypes/
│   │   ├── EffectType_Weapon_Impacts.uasset
│   │   ├── EffectType_World_Burst.uasset
│   │   ├── EffectType_World_Placeable.uasset
│   │   └── EffectType_World_Portal.uasset
│   ├── MaterialFunctions/
│   │   ├── MF_AlphaLevels.uasset
│   │   ├── MF_AngleFade.uasset
│   │   ├── MF_AnimatedSquareTexture.uasset
│   │   ├── MF_ChannelRemap.uasset
│   │   ├── MF_ChannelRemapRGB.uasset
│   │   ├── MF_DepthCheck.uasset
│   │   ├── MF_DepthFade.uasset
│   │   ├── MF_FastSphereNormal.uasset
│   │   ├── MF_MannequinEdge.uasset
│   │   ├── MF_MannequinHitLocations.uasset
│   │   ├── MF_MannequinHitMacro.uasset
│   │   ├── MF_MannequinLaunchEdgeGlow.uasset
│   │   ├── MF_MannequinWPODeform.uasset
│   │   ├── MF_MannequinWoundBlend.uasset
│   │   ├── MF_MannequinWoundMaterial.uasset
│   │   ├── MF_MotionStretch.uasset
│   │   ├── MF_NearDistanceFade.uasset
│   │   ├── MF_ParticleOffsetScreenUV.uasset
│   │   ├── MF_PortalEdgeTexture.uasset
│   │   ├── MF_RotateVector2.uasset
│   │   ├── MF_SpriteDiffuse.uasset
│   │   ├── MF_SpriteEmissive.uasset
│   │   ├── MF_SpriteNormal.uasset
│   │   ├── MF_SpriteOpacity.uasset
│   │   ├── MF_SpriteParallax.uasset
│   │   ├── MF_SpriteUV.uasset
│   │   ├── MF_TextureChannelSelect.uasset
│   │   ├── MF_TextureDiffuse.uasset
│   │   ├── MF_TextureEmissive.uasset
│   │   ├── MF_TextureOpacity.uasset
│   │   ├── MF_TextureParallax.uasset
│   │   ├── MF_TracerSmokeOpacity.uasset
│   │   ├── MF_TrigSphereNormal.uasset
│   │   ├── MF_teleporterUVMask.uasset
│   │   └── MF_teleporterUVMaskSimple.uasset
│   ├── Materials/
│   │   ├── BlastGraphics/
│   │   │   ├── M_Grenade_Blast.uasset
│   │   │   ├── M_Grenade_EX_Flare.uasset
│   │   │   ├── M_Grenade_GraphicsTrail_1.uasset
│   │   │   ├── M_Grenade_GraphicsTrail_2.uasset
│   │   │   ├── M_Grenade_GraphicsTrail_3.uasset
│   │   │   ├── M_ParticleBlast.uasset
│   │   │   └── M_ParticleBlastDisc.uasset
│   │   ├── Cubes/
│   │   │   ├── Instances/
│   │   │   │   └── MI_SpriteCubes.uasset
│   │   │   ├── M_Character_Cubes.uasset
│   │   │   ├── M_Portal_Cubes.uasset
│   │   │   └── M_SpriteCubes.uasset
│   │   ├── DamageNumbers/
│   │   │   ├── Instances/
│   │   │   │   ├── MI_numba.uasset
│   │   │   │   └── MI_numberSingleCard.uasset
│   │   │   └── M_numba.uasset
│   │   ├── Dash/
│   │   │   ├── M_DuplicateRibbon.uasset
│   │   │   ├── M_RefractiveSprite.uasset
│   │   │   └── M_launchpadRibbon.uasset
│   │   ├── Decals/
│   │   │   ├── Instances/
│   │   │   │   ├── MI_Decal_Concrete.uasset
│   │   │   │   └── MI_Decal_Transparent_Glass.uasset
│   │   │   ├── M_Decal_Base.uasset
│   │   │   ├── M_Decal_Transparent_Base.uasset
│   │   │   ├── M_ExplosionDecal.uasset
│   │   │   ├── M_GlassBulletHoles_Decal.uasset
│   │   │   ├── M_Honetcomb_BulletHit_Decal.uasset
│   │   │   └── M_HoneyCombDecal.uasset
│   │   ├── FootSteps/
│   │   │   ├── Instances/
│   │   │   │   ├── MI_Flipbook_Pyro_PuffSmoke.uasset
│   │   │   │   ├── MI_FootStepDecal_All.uasset
│   │   │   │   └── MI_FootStepDecal_Glass.uasset
│   │   │   ├── M_Flipbook_Pyro_Lit.uasset
│   │   │   ├── M_FootStepDecal_Base.uasset
│   │   │   └── M_FootStepDecal_Debug.uasset
│   │   ├── General/
│   │   │   ├── Instances/
│   │   │   │   └── MI_ParticleColor.uasset
│   │   │   ├── M_ParticleDebug.uasset
│   │   │   └── M_PartileColor.uasset
│   │   ├── Meshes/
│   │   │   ├── M_BasicBulletCasing.uasset
│   │   │   ├── M_BasicShotgunBulletCasing.uasset
│   │   │   ├── M_CF_boxes.uasset
│   │   │   ├── M_CF_rings.uasset
│   │   │   ├── M_DebrisChunks_1.uasset
│   │   │   ├── M_DebrisChunks_2.uasset
│   │   │   ├── M_DebrisChunks_3.uasset
│   │   │   ├── M_FX_Hardsurface_Masked.uasset
│   │   │   └── M_FX_Hardsurface_Transparent.uasset
│   │   ├── MuzzleFlash/
│   │   │   ├── Instances/
│   │   │   │   ├── MI_Flipbook_MuzzleFlash_Front.uasset
│   │   │   │   ├── MI_Flipbook_MuzzleFlash_Side.uasset
│   │   │   │   └── MI_Flipbook_MuzzleFlash_Top.uasset
│   │   │   └── M_Flipbook_MuzzleFlash_Base.uasset
│   │   ├── PickUp/
│   │   │   ├── Instances/
│   │   │   │   └── M_GunPickup_RingMeshNoFade.uasset
│   │   │   ├── M_GunPad_Pixels.uasset
│   │   │   ├── M_GunPad_Trails.uasset
│   │   │   ├── M_GunPickup_RingMesh.uasset
│   │   │   ├── M_GunPickup_Rings.uasset
│   │   │   ├── M_Pad_Beam.uasset
│   │   │   ├── M_Pad_Graphics_Diffus_Core.uasset
│   │   │   └── M_Pads_CubeTrails.uasset
│   │   ├── PostProcess/
│   │   │   ├── MPP_Outline.uasset
│   │   │   └── MPP_Outline_Inst.uasset
│   │   ├── Pyro/
│   │   │   ├── Instances/
│   │   │   │   ├── MI_Flipbook_Pyro_Default.uasset
│   │   │   │   ├── MI_Flipbook_Pyro_Diffuse.uasset
│   │   │   │   ├── MI_Flipbook_Pyro_Diffuse_ImpactSmoke.uasset
│   │   │   │   ├── MI_Flipbook_Pyro_Diffuse_MuzzleFlashSmoke.uasset
│   │   │   │   ├── MI_Flipbook_Pyro_Emissive.uasset
│   │   │   │   └── MI_Flipbook_Pyro_Grenade.uasset
│   │   │   └── M_Flipbook_Pyro_Base.uasset
│   │   ├── Sparks/
│   │   │   ├── Instances/
│   │   │   │   ├── MI_Sparks_Dust.uasset
│   │   │   │   └── MI_Sparks_Metal.uasset
│   │   │   └── M_Sparks_Base.uasset
│   │   ├── Teleporter/
│   │   │   ├── Instances/
│   │   │   │   ├── MI_portalBGParticle.uasset
│   │   │   │   ├── MI_portalBGParticleTriangle.uasset
│   │   │   │   ├── MI_portalInteriorShape.uasset
│   │   │   │   └── MI_portalSprite.uasset
│   │   │   ├── M_portalBGParticle.uasset
│   │   │   ├── M_portalBGParticleOpaque.uasset
│   │   │   ├── M_portalInteriorShape.uasset
│   │   │   └── M_portalSprite.uasset
│   │   └── Tracers/
│   │       ├── Instances/
│   │       │   ├── MI_TracerSmoke_Default.uasset
│   │       │   └── MI_Tracers_Default.uasset
│   │       ├── M_TracerCore_Base.uasset
│   │       └── M_TracerSmoke_Base.uasset
│   ├── Meshes/
│   │   ├── BulletShells/
│   │   │   ├── SM_pistolshell.uasset
│   │   │   ├── SM_rifleshell.uasset
│   │   │   └── SM_shotgunshell.uasset
│   │   ├── Portal/
│   │   │   └── portal_square_unit.uasset
│   │   ├── Rocks/
│   │   │   ├── Debris_Chunk1.uasset
│   │   │   ├── Debris_Chunk2.uasset
│   │   │   ├── Debris_Chunk3.uasset
│   │   │   └── ultkedqjw_LOD7.uasset
│   │   ├── Cube.uasset
│   │   ├── box.uasset
│   │   └── ring.uasset
│   ├── NiagaraModules/
│   │   ├── NM_BPSystemEvent.uasset
│   │   ├── NM_BPSystemEventSpawnMask.uasset
│   │   ├── NM_FootPrintSizeAndOrientation.uasset
│   │   ├── NM_ImpactReadDataChannel.uasset
│   │   ├── NM_ImpactSpawnAttributes.uasset
│   │   ├── NM_ImpactSpawnDataChannel.uasset
│   │   ├── NM_ImpactSpawnDataChannelConditional.uasset
│   │   ├── NM_MotionStretch.uasset
│   │   ├── NM_OffsetPositionByVector.uasset
│   │   ├── NM_ParticleLight.uasset
│   │   ├── NM_ProjectToPlane.uasset
│   │   ├── NM_SkeletalMeshReferencePosition.uasset
│   │   ├── NM_SubImageIndexVariants.uasset
│   │   └── NM_WeaponTrigger.uasset
│   ├── Particles/
│   │   ├── Environmental/
│   │   │   ├── Emitters/
│   │   │   │   ├── NE_BoneBasedEmission.uasset
│   │   │   │   └── NE_SkeletalSurfaceBasedEmission.uasset
│   │   │   ├── NS_CharacterDash.uasset
│   │   │   ├── NS_CharacterPortalDissolve.uasset
│   │   │   ├── NS_CharacterSpawnIn.uasset
│   │   │   ├── NS_CharacterSpawnIn2.uasset
│   │   │   ├── NS_ElectricMovement.uasset
│   │   │   ├── NS_JumpPad.uasset
│   │   │   ├── NS_WallPortal.uasset
│   │   │   └── NS_WallPortalEnter.uasset
│   │   ├── Explosion/
│   │   │   ├── NS_Grenade_Explosion.uasset
│   │   │   └── NS_Grenade_Trail.uasset
│   │   ├── Footsteps/
│   │   │   └── NS_Footsteps.uasset
│   │   ├── Impacts/
│   │   │   ├── Emitters/
│   │   │   │   ├── NE_DustyFlipbook.uasset
│   │   │   │   ├── NE_ImpactCone.uasset
│   │   │   │   ├── NE_ImpactCore.uasset
│   │   │   │   ├── NE_ImpactParticleLight.uasset
│   │   │   │   ├── NE_ImpactSparks.uasset
│   │   │   │   └── NE_Rocks.uasset
│   │   │   ├── NS_DamageNumbers.uasset
│   │   │   ├── NS_DeathCubes.uasset
│   │   │   ├── NS_ImactSparksCharacter.uasset
│   │   │   ├── NS_ImpactConcrete.uasset
│   │   │   ├── NS_ImpactDataChannel.uasset
│   │   │   ├── NS_ImpactDecals.uasset
│   │   │   └── NS_ImpactGlass.uasset
│   │   ├── Item/
│   │   │   ├── Emitters/
│   │   │   │   └── NE_RingMeshTimer.uasset
│   │   │   ├── NS_GunPad.uasset
│   │   │   ├── NS_GunPad_Loading.uasset
│   │   │   ├── NS_GunPad_Pickup.uasset
│   │   │   └── NS_Heal.uasset
│   │   ├── Lighting/
│   │   │   └── Emitters/
│   │   │       └── NE_ParticleLight.uasset
│   │   └── Weapons/
│   │       ├── Emitters/
│   │       │   ├── NE_MuzzleFlashCards.uasset
│   │       │   ├── NE_MuzzleFlashFlipbookCards.uasset
│   │       │   ├── NE_MuzzleFlashSmoke.uasset
│   │       │   ├── NE_MuzzleFlashSmokePuff.uasset
│   │       │   ├── NE_MuzzleFlashSparks.uasset
│   │       │   ├── NE_MuzzleFlashStarBurst.uasset
│   │       │   ├── NE_MuzzleFlashStarburstFlipbookCards.uasset
│   │       │   ├── N_Emitter_Bullet_Impact.uasset
│   │       │   └── N_Emitter_MuzzleFlash.uasset
│   │       ├── NS_WeaponFire.uasset
│   │       ├── NS_WeaponFire_MuzzleFlash_Rifle.uasset
│   │       ├── NS_WeaponFire_ShellEject.uasset
│   │       ├── NS_WeaponFire_Tracer.uasset
│   │       └── NS_WeaponFire_Tracer_Shotgun.uasset
│   ├── Physics/
│   │   ├── B_ExamplePhysicsField.uasset
│   │   ├── B_ShatterSphere.uasset
│   │   ├── M_GlowyPink.uasset
│   │   └── ShatterSphere_GeometryCollection.uasset
│   └── Textures/
│       ├── BlastGraphics/
│       │   ├── T_ExplosionFlare.uasset
│       │   └── T_Graphics.uasset
│       ├── Cutouts/
│       │   ├── T_Cutout_Octagon_Mask.uasset
│       │   └── T_cutout_gradient_vertical.uasset
│       ├── Decals/
│       │   ├── T_ExplosionDecal.uasset
│       │   ├── T_ExplsionDigital.uasset
│       │   ├── chippedcracks.uasset
│       │   ├── concrete_depth2x2.uasset
│       │   ├── concrete_normal2x2.uasset
│       │   ├── glass_mask2x2.uasset
│       │   ├── glass_normal2x2.uasset
│       │   └── hexagon.uasset
│       ├── Flipbooks/
│       │   ├── LDR_Squib_2_rop_comp1_0001_flipbook.uasset
│       │   ├── SmokeSwirl_3_Flipbook_CHANNELPACK.uasset
│       │   ├── T_Explosion_Areal_E64.uasset
│       │   ├── numbers_color.uasset
│       │   ├── numbers_normal.uasset
│       │   ├── rotatingcube_sprite_Cd_0.uasset
│       │   └── rotatingcube_sprite_N_0.uasset
│       ├── FootPrint/
│       │   ├── T_FootPrint_01_MSK.uasset
│       │   ├── T_FootPrint_01_N.uasset
│       │   └── T_FootStep_Decal.uasset
│       ├── General/
│       │   ├── Good64x64TilingNoiseHighFreq.uasset
│       │   ├── T_Cube_BW.uasset
│       │   ├── T_Inky_Smoke_Tile.uasset
│       │   ├── T_NanoFiber_Pattern_N.uasset
│       │   ├── blurry_texture.uasset
│       │   ├── squares.uasset
│       │   └── textureNoise.uasset
│       ├── LaunchPad/
│       │   ├── launchbar_mask.uasset
│       │   └── launchbar_normal.uasset
│       ├── MuzzleFlash/
│       │   └── T_MuzzleFlash_Directional_01_MSK.uasset
│       ├── Rocks/
│       │   ├── Debris_Chunk1_BaseColor.uasset
│       │   ├── Debris_Chunk2_BaseColor.uasset
│       │   ├── Debris_Chunk3_BaseColor.uasset
│       │   ├── WhiteSquareTexture.uasset
│       │   ├── ultkedqjw_4K_Albedo.uasset
│       │   ├── ultkedqjw_4K_Normal_LOD0.uasset
│       │   └── ultkedqjw_4K_Roughness.uasset
│       └── Teleporter/
│           └── rounderCorner.uasset
├── Environments/
│   ├── Gameplay/
│   │   ├── AS_InstantHeal.uasset
│   │   ├── BP_GameplayEffectPad.uasset
│   │   ├── B_GameplayEffectPad_Child_Healing.uasset
│   │   ├── GE_GameplayEffectPad_Damage.uasset
│   │   ├── GE_GameplayEffectPad_Heal.uasset
│   │   └── M_AdjustablePadColor.uasset
│   ├── Materials/
│   │   └── Grid/
│   │       └── TextureGrid/
│   │           ├── Grid_BW_512.uasset
│   │           ├── MI_Grid_Texture.uasset
│   │           └── M_Grid_Texture.uasset
│   └── B_LoadRandomLobbyBackground.uasset
├── Feedback/
│   ├── CameraShakes/
│   │   ├── CS_Character_DamageTaken.uasset
│   │   ├── CS_Character_Heal.uasset
│   │   ├── CS_Weapon_Fire.uasset
│   │   ├── CS_Weapon_Fire_Pistol.uasset
│   │   ├── CS_Weapon_Fire_Rifle.uasset
│   │   └── CS_Weapon_Fire_Shotgun.uasset
│   └── Haptics/
│       ├── Weapon_Fire_Auto/
│       │   ├── C_Weapon_AutoVibration_Amplitude.uasset
│       │   ├── C_Weapon_AutoVibration_Frequency.uasset
│       │   ├── C_Weapon_AutoVibration_Position.uasset
│       │   └── FFE_Weapon_Fire_Auto.uasset
│       ├── FFE_Character_Damage.uasset
│       ├── FFE_Character_Heal.uasset
│       └── FFE_Weapon_Fire.uasset
├── GameplayCueNotifies/
│   ├── GCNL_Character_DamageTaken.uasset
│   ├── GCNL_Test_Looping.uasset
│   ├── GCNL_Widget_Base.uasset
│   ├── GCN_Character_Heal.uasset
│   ├── GCN_Test_Burst.uasset
│   ├── GCN_Test_BurstLatent.uasset
│   ├── GCN_Weapon_Impact.uasset
│   └── I_GameplayCueWidget.uasset
├── GameplayEffects/
│   ├── Damage/
│   │   ├── GE_Damage_Basic_Instant.uasset
│   │   ├── GE_Damage_Basic_Periodic.uasset
│   │   ├── GE_Damage_Basic_SetByCaller.uasset
│   │   └── GameplayEffectParent_Damage_Basic.uasset
│   ├── Heal/
│   │   ├── GE_Heal_Instant.uasset
│   │   ├── GE_Heal_Periodic.uasset
│   │   ├── GE_Heal_SetByCaller.uasset
│   │   └── GameplayEffectParent_Heal.uasset
│   ├── GE_BlockAbilityInput.uasset
│   ├── GE_DynamicTag.uasset
│   ├── GE_GameplayCueTest_Burst.uasset
│   ├── GE_GameplayCueTest_BurstLatent.uasset
│   ├── GE_GameplayCueTest_Looping.uasset
│   ├── GE_HeroDash_Cooldown.uasset
│   └── GE_IsPlayer.uasset
├── Input/
│   ├── Actions/
│   │   ├── IA_Ability_Dash.uasset
│   │   ├── IA_Ability_Heal.uasset
│   │   ├── IA_AutoRun.uasset
│   │   ├── IA_Crouch.uasset
│   │   ├── IA_Jump.uasset
│   │   ├── IA_Look_Mouse.uasset
│   │   ├── IA_Look_Stick.uasset
│   │   ├── IA_Move.uasset
│   │   ├── IA_Weapon_Fire.uasset
│   │   ├── IA_Weapon_Fire_Auto.uasset
│   │   └── IA_Weapon_Reload.uasset
│   ├── Mappings/
│   │   └── IMC_Default.uasset
│   ├── Settings/
│   │   └── GamepadAimSensitivity_Normal.uasset
│   ├── InputData_Hero.uasset
│   └── InputData_SimplePawn.uasset
├── Legal/
│   └── ThirdParty/
│       └── Licenses/
├── Localization/
│   ├── EngineOverrides/
│   │   ├── ar/
│   │   ├── de/
│   │   ├── en/
│   │   ├── es/
│   │   ├── es-419/
│   │   ├── fr/
│   │   ├── it/
│   │   ├── ja/
│   │   ├── ko/
│   │   ├── pl/
│   │   ├── pt-BR/
│   │   ├── ru/
│   │   ├── tr/
│   │   ├── zh-Hans/
│   └── Game/
│       ├── ar/
│       ├── de/
│       ├── en/
│       ├── es/
│       ├── es-419/
│       ├── fr/
│       ├── it/
│       ├── ja/
│       ├── ko/
│       ├── pl/
│       ├── pt-BR/
│       ├── ru/
│       ├── tr/
│       ├── zh-Hans/
├── PhysicsMaterials/
│   ├── PM_Character.uasset
│   ├── PM_Concrete.uasset
│   └── PM_Glass.uasset
├── System/
│   ├── DefaultEditorMap/
│   │   ├── B_ExperienceList3D.uasset
│   │   ├── B_TeleportToUserFacingExperience.uasset
│   │   ├── L_DefaultEditorOverview.umap
│   │   └── L_DefaultEditorOverview_BuiltData.uasset
│   ├── Experiences/
│   │   └── B_LyraDefaultExperience.uasset
│   ├── FrontEnd/
│   │   ├── Maps/
│   │   │   ├── L_LyraFrontEnd.umap
│   │   │   └── L_LyraFrontEnd_BuiltData.uasset
│   │   ├── B_LyraFrontEnd_Experience.uasset
│   │   └── T_UI_BackdropPlaceholder_Mat.uasset
│   ├── Playlists/
│   │   ├── DA_ExamplePlaylist.uasset
│   │   └── DA_Frontend.uasset
│   ├── Teams/
│   │   ├── TeamDA_Blue.uasset
│   │   ├── TeamDA_Green.uasset
│   │   ├── TeamDA_Red.uasset
│   │   └── TeamDA_Yellow.uasset
│   └── TransitionMap.umap
├── Tools/
│   ├── BakedGeneratedMeshSystem/
│   │   ├── BaseClasses/
│   │   │   ├── BakedGeneratedMeshActor.uasset
│   │   │   ├── BakedStaticMeshActor.uasset
│   │   │   └── Enum_BakedGeneratedState.uasset
│   │   ├── EditorActions/
│   │   │   ├── FindSourceMesh.uasset
│   │   │   ├── SwapGeneratedActor_FromSM.uasset
│   │   │   ├── SwapGeneratedActor_ToSM.uasset
│   │   │   └── SyncSourceKey.uasset
│   │   ├── BakableTwistyBoxDemo.uasset
│   │   └── GeneratedMeshColdStorage.uasset
│   ├── Materials/
│   │   └── MT_DebugTool.uasset
│   ├── PanelPieces/
│   │   ├── SM_PanelFrameCurveComponents_1_PanelFrameStraightA.uasset
│   │   ├── SM_PanelFrameCurveComponents_3_PanelFrameCurveA.uasset
│   │   ├── SM_PanelFrameCurveComponents_3_Panel_1x1_B_TwoSidedBevel.uasset
│   │   ├── SM_PanelImport_Panel_1x1_A_NoBevel.uasset
│   │   ├── SM_PanelImport_Panel_1x1_A_OneSidedBevel.uasset
│   │   ├── SM_PanelImport_Panel_1x1_A_TwoSidedBevel.uasset
│   │   └── SM_PanelImport_Panel_1x1_C_TwoSidedBevel.uasset
│   ├── B_GeneratedTube.uasset
│   ├── B_GeneratedTube_Advanced.uasset
│   ├── B_Tool_AdvancedWindow.uasset
│   ├── B_Tool_CornerExtrude.uasset
│   ├── B_Tool_Panel_BGM.uasset
│   ├── B_Tool_RampMakerControl_BGM.uasset
│   ├── B_Tool_Repeater.uasset
│   ├── B_Tool_Stairs_BGM.uasset
│   ├── B_WindowDoor.uasset
│   ├── Enum_PanelType.uasset
│   ├── Enum_RailingConstructionType.uasset
│   ├── Enum_RotationAxis.uasset
│   ├── Enum_StairConstructionType.uasset
│   ├── Enum_WindowType.uasset
│   ├── MT_PanelCenterDefault.uasset
│   ├── MT_PanelCornerDefault.uasset
│   ├── MT_RailingDefaultMaterial.uasset
│   ├── MT_StairDefaultMaterial.uasset
│   ├── SM_StairStep.uasset
│   └── SM_StairStep100.uasset
├── UI/
│   ├── Credits/
│   │   ├── W_Credits.uasset
│   │   └── W_TapToggleActionBar.uasset
│   ├── Foundation/
│   │   ├── Buttons/
│   │   │   ├── ButtonStyle-Clear.uasset
│   │   │   ├── ButtonStyle-Primary-M.uasset
│   │   │   ├── W_LyraButton.uasset
│   │   │   └── W_LyraButtonTab.uasset
│   │   ├── Dialogs/
│   │   │   ├── W_ConfirmationDefault.uasset
│   │   │   ├── W_ConfirmationDialog.uasset
│   │   │   ├── W_ConfirmationError.uasset
│   │   │   └── W_ControllerDisconnected.uasset
│   │   ├── Fonts/
│   │   │   ├── NotoSans/
│   │   │   │   ├── NotoColorEmoji.uasset
│   │   │   │   ├── NotoSans-Black.uasset
│   │   │   │   ├── NotoSans-Bold.uasset
│   │   │   │   ├── NotoSans-Light.uasset
│   │   │   │   ├── NotoSans-Regular.uasset
│   │   │   │   ├── NotoSansArabic-Bold.uasset
│   │   │   │   ├── NotoSansArabic-Regular.uasset
│   │   │   │   ├── NotoSansJP-Bold.uasset
│   │   │   │   ├── NotoSansKR-Regular.uasset
│   │   │   │   ├── NotoSansSC-Regular.uasset
│   │   │   │   └── NotoSansTC-Regular.uasset
│   │   │   ├── Orbitron/
│   │   │   │   ├── Raw/
│   │   │   │   ├── Orbitron-Black.uasset
│   │   │   │   ├── Orbitron-Bold.uasset
│   │   │   │   └── Orbitron-Medium.uasset
│   │   │   ├── NotoSans.uasset
│   │   │   └── Orbitron.uasset
│   │   ├── Icons/
│   │   │   └── T-Icon-Warning-64.uasset
│   │   ├── LoadingScreen/
│   │   │   ├── W_LoadingScreenReasonDebugText.uasset
│   │   │   ├── W_LoadingScreen_DefaultContent.uasset
│   │   │   ├── W_LoadingScreen_Host.uasset
│   │   │   ├── W_LyraLogo_LoadingScreen.uasset
│   │   │   └── W_LyraLogo_Small.uasset
│   │   ├── Materials/
│   │   │   ├── BlueButton/
│   │   │   │   ├── Button_Base_Blue.uasset
│   │   │   │   ├── Button_Default_Disabled.uasset
│   │   │   │   ├── Button_Default_Hovered.uasset
│   │   │   │   └── Button_Default_Hovered_Selected.uasset
│   │   │   ├── Functions/
│   │   │   │   ├── MF_UI_AngleGradient.uasset
│   │   │   │   ├── MF_UI_Arc.uasset
│   │   │   │   ├── MF_UI_CursorPosition.uasset
│   │   │   │   ├── MF_UI_RadialSegments.uasset
│   │   │   │   ├── MF_UI_SquareToScreenSpace.uasset
│   │   │   │   └── MF_UI_Viewport_UI_Scale.uasset
│   │   │   ├── Textures/
│   │   │   │   ├── T-UI-Shine.uasset
│   │   │   │   └── T-UI-TestIcon.uasset
│   │   │   ├── MI_UI_Button_Base.uasset
│   │   │   ├── M_UI_AngledBox_Base.uasset
│   │   │   ├── M_UI_FocusBorderPulse_Base.uasset
│   │   │   ├── M_UI_FocusBorderPulse_Circle_Base.uasset
│   │   │   ├── M_UI_HollowCircularProgressBar_Base.uasset
│   │   │   ├── M_UI_IconGlow_Base.uasset
│   │   │   ├── M_UI_RadialGradient_Base.uasset
│   │   │   ├── M_UI_RadialProgress.uasset
│   │   │   ├── M_UI_ShineOutline_Base.uasset
│   │   │   └── M_UI_Throbber_Base.uasset
│   │   ├── Platform/
│   │   │   └── Input/
│   │   │       ├── GamepadPS4/
│   │   │       │   ├── CommonInput_Gamepad_PS4.uasset
│   │   │       │   ├── ControllerConfig.uasset
│   │   │       │   ├── T_PS4_Circle.uasset
│   │   │       │   ├── T_PS4_Cross.uasset
│   │   │       │   ├── T_PS4_DPad.uasset
│   │   │       │   ├── T_PS4_DPad_Down.uasset
│   │   │       │   ├── T_PS4_DPad_Left.uasset
│   │   │       │   ├── T_PS4_DPad_Right.uasset
│   │   │       │   ├── T_PS4_DPad_Up.uasset
│   │   │       │   ├── T_PS4_Dpad_LeftRight.uasset
│   │   │       │   ├── T_PS4_Dpad_UpDown.uasset
│   │   │       │   ├── T_PS4_L1.uasset
│   │   │       │   ├── T_PS4_L2.uasset
│   │   │       │   ├── T_PS4_LStick.uasset
│   │   │       │   ├── T_PS4_LStick_Down.uasset
│   │   │       │   ├── T_PS4_LStick_Left.uasset
│   │   │       │   ├── T_PS4_LStick_Pan.uasset
│   │   │       │   ├── T_PS4_LStick_Press.uasset
│   │   │       │   ├── T_PS4_LStick_Right.uasset
│   │   │       │   ├── T_PS4_LStick_Scroll.uasset
│   │   │       │   ├── T_PS4_LStick_Up.uasset
│   │   │       │   ├── T_PS4_Options.uasset
│   │   │       │   ├── T_PS4_R1.uasset
│   │   │       │   ├── T_PS4_R2.uasset
│   │   │       │   ├── T_PS4_RStick.uasset
│   │   │       │   ├── T_PS4_RStick_Down.uasset
│   │   │       │   ├── T_PS4_RStick_Left.uasset
│   │   │       │   ├── T_PS4_RStick_Pan.uasset
│   │   │       │   ├── T_PS4_RStick_Press.uasset
│   │   │       │   ├── T_PS4_RStick_Right.uasset
│   │   │       │   ├── T_PS4_RStick_Scroll.uasset
│   │   │       │   ├── T_PS4_RStick_Up.uasset
│   │   │       │   ├── T_PS4_Share.uasset
│   │   │       │   ├── T_PS4_Square.uasset
│   │   │       │   ├── T_PS4_TouchPad.uasset
│   │   │       │   └── T_PS4_Triangle.uasset
│   │   │       ├── GamepadPS5/
│   │   │       │   ├── CommonInput_Gamepad_PS5.uasset
│   │   │       │   └── ControllerConfig.uasset
│   │   │       ├── GamepadSwitch/
│   │   │       │   ├── CommonInput_Gamepad_Switch.uasset
│   │   │       │   ├── ControllerConfig.uasset
│   │   │       │   ├── T_NS_A.uasset
│   │   │       │   ├── T_NS_B.uasset
│   │   │       │   ├── T_NS_Capture.uasset
│   │   │       │   ├── T_NS_DPad.uasset
│   │   │       │   ├── T_NS_DPad_Down.uasset
│   │   │       │   ├── T_NS_DPad_Left.uasset
│   │   │       │   ├── T_NS_DPad_LeftRight.uasset
│   │   │       │   ├── T_NS_DPad_Right.uasset
│   │   │       │   ├── T_NS_DPad_Up.uasset
│   │   │       │   ├── T_NS_DPad_UpDown.uasset
│   │   │       │   ├── T_NS_Home.uasset
│   │   │       │   ├── T_NS_LB.uasset
│   │   │       │   ├── T_NS_LStick.uasset
│   │   │       │   ├── T_NS_LStick_Down.uasset
│   │   │       │   ├── T_NS_LStick_Left.uasset
│   │   │       │   ├── T_NS_LStick_Pan.uasset
│   │   │       │   ├── T_NS_LStick_Press.uasset
│   │   │       │   ├── T_NS_LStick_Right.uasset
│   │   │       │   ├── T_NS_LStick_Scroll.uasset
│   │   │       │   ├── T_NS_LStick_Up.uasset
│   │   │       │   ├── T_NS_Minus.uasset
│   │   │       │   ├── T_NS_Plus.uasset
│   │   │       │   ├── T_NS_R.uasset
│   │   │       │   ├── T_NS_RStick.uasset
│   │   │       │   ├── T_NS_RStick_Down.uasset
│   │   │       │   ├── T_NS_RStick_Left.uasset
│   │   │       │   ├── T_NS_RStick_Pan.uasset
│   │   │       │   ├── T_NS_RStick_Press.uasset
│   │   │       │   ├── T_NS_RStick_Right.uasset
│   │   │       │   ├── T_NS_RStick_Scroll.uasset
│   │   │       │   ├── T_NS_RStick_Up.uasset
│   │   │       │   ├── T_NS_X.uasset
│   │   │       │   ├── T_NS_Y.uasset
│   │   │       │   ├── T_NS_ZL.uasset
│   │   │       │   └── T_NS_ZR.uasset
│   │   │       ├── GamepadXboxOne/
│   │   │       │   ├── CommonInput_Gamepad_XboxOne.uasset
│   │   │       │   ├── ControllerConfig.uasset
│   │   │       │   ├── T_XB1_A.uasset
│   │   │       │   ├── T_XB1_B.uasset
│   │   │       │   ├── T_XB1_DPad.uasset
│   │   │       │   ├── T_XB1_DPad_Down.uasset
│   │   │       │   ├── T_XB1_DPad_Left.uasset
│   │   │       │   ├── T_XB1_DPad_Right.uasset
│   │   │       │   ├── T_XB1_DPad_Up.uasset
│   │   │       │   ├── T_XB1_DPad_UpDown.uasset
│   │   │       │   ├── T_XB1_Dpad_LeftRight.uasset
│   │   │       │   ├── T_XB1_LB.uasset
│   │   │       │   ├── T_XB1_LStick.uasset
│   │   │       │   ├── T_XB1_LStick_Down.uasset
│   │   │       │   ├── T_XB1_LStick_Left.uasset
│   │   │       │   ├── T_XB1_LStick_Pan.uasset
│   │   │       │   ├── T_XB1_LStick_Press.uasset
│   │   │       │   ├── T_XB1_LStick_Right.uasset
│   │   │       │   ├── T_XB1_LStick_Scroll.uasset
│   │   │       │   ├── T_XB1_LStick_Up.uasset
│   │   │       │   ├── T_XB1_LT.uasset
│   │   │       │   ├── T_XB1_Menu.uasset
│   │   │       │   ├── T_XB1_RB.uasset
│   │   │       │   ├── T_XB1_RStick.uasset
│   │   │       │   ├── T_XB1_RStick_Down.uasset
│   │   │       │   ├── T_XB1_RStick_Left.uasset
│   │   │       │   ├── T_XB1_RStick_Pan.uasset
│   │   │       │   ├── T_XB1_RStick_Press.uasset
│   │   │       │   ├── T_XB1_RStick_Right.uasset
│   │   │       │   ├── T_XB1_RStick_Scroll.uasset
│   │   │       │   ├── T_XB1_RStick_Up.uasset
│   │   │       │   ├── T_XB1_RT.uasset
│   │   │       │   ├── T_XB1_Windows.uasset
│   │   │       │   ├── T_XB1_X.uasset
│   │   │       │   └── T_XB1_Y.uasset
│   │   │       ├── GamepadXboxSeriesX/
│   │   │       │   ├── CommonInput_Gamepad_XSX.uasset
│   │   │       │   ├── ControllerConfig.uasset
│   │   │       │   └── T_XB1_Share.uasset
│   │   │       ├── KeyboardMouse/
│   │   │       │   ├── CommonInput_KeyboardMouse.uasset
│   │   │       │   ├── Mouse1_T.uasset
│   │   │       │   ├── Mouse2_T.uasset
│   │   │       │   ├── Mouse3_T.uasset
│   │   │       │   ├── Mouse4_T.uasset
│   │   │       │   ├── Mouse5_T.uasset
│   │   │       │   ├── MouseAxis-X.uasset
│   │   │       │   ├── MouseScrollAxis_T.uasset
│   │   │       │   ├── MouseWheelDown_T.uasset
│   │   │       │   └── MouseWheelUp_T.uasset
│   │   │       └── Misc/
│   │   │           ├── Keybind-Frame-L.uasset
│   │   │           ├── Keybind-Frame-M.uasset
│   │   │           └── Keybind-Frame-S.uasset
│   │   ├── RichTextData/
│   │   │   ├── CommonUIRichTextData.uasset
│   │   │   └── RichTextIconSet.uasset
│   │   ├── SoftwareCursors/
│   │   │   ├── CursorArrow.uasset
│   │   │   └── W_ArrowCursor.uasset
│   │   ├── Subtitles/
│   │   │   └── W_SubtitleDisplayHost.uasset
│   │   ├── TabbedView/
│   │   │   └── W_HorizontalTabList.uasset
│   │   ├── Text/
│   │   │   ├── Materials/
│   │   │   │   └── M_Font_SimpleGradient.uasset
│   │   │   ├── TextScrollStyle-Base.uasset
│   │   │   ├── TextStyle-Base.uasset
│   │   │   ├── TextStyle-BaseParent.uasset
│   │   │   ├── TextStyle-Huge.uasset
│   │   │   ├── TextStyle-Regular-Disabled.uasset
│   │   │   ├── TextStyle-Regular-WithShadow.uasset
│   │   │   ├── TextStyle-Regular.uasset
│   │   │   ├── TextStyle-Small++.uasset
│   │   │   ├── TextStyle-Small+.uasset
│   │   │   └── TextStyle-Small.uasset
│   │   └── Widgets/
│   │       ├── BottomBar/
│   │       │   ├── W_BottomActionBar.uasset
│   │       │   └── W_BoundActionButton.uasset
│   │       ├── SimpleProgressBar/
│   │       │   ├── Materials/
│   │       │   │   ├── M_ProgressBar_Segmented.uasset
│   │       │   │   ├── M_ProgressBar_Segmented_Inst.uasset
│   │       │   │   ├── M_ProgressBar_Segmented_NoStroke.uasset
│   │       │   │   ├── T_UI_ReticleSegmentGradient.uasset
│   │       │   │   └── T_UI_Square_DF.uasset
│   │       │   └── W_SimpleProgressBar.uasset
│   │       ├── InputActionWidget.uasset
│   │       ├── W_AspectImageWithBlur.uasset
│   │       └── W_ProgressSpinner.uasset
│   ├── FrontEnd/
│   │   └── W_FrontEndHUDLayout.uasset
│   ├── Hud/
│   │   ├── Art/
│   │   │   ├── CA_UI_HUD.uasset
│   │   │   ├── C_UI_Accolade_Alpha.uasset
│   │   │   ├── C_UI_Accolade_Color.uasset
│   │   │   ├── C_UI_AmmoCounter.uasset
│   │   │   ├── C_UI_ControlRing.uasset
│   │   │   ├── C_UI_DashCooldown.uasset
│   │   │   ├── C_UI_ElimFeed_Glow-Shadow.uasset
│   │   │   ├── C_UI_Elimination_Alpha.uasset
│   │   │   ├── C_UI_Elimination_Color.uasset
│   │   │   ├── C_UI_HealthBar_Shadow-Notch-Glow.uasset
│   │   │   ├── C_UI_TeamScore.uasset
│   │   │   ├── C_UI_WeaponCard.uasset
│   │   │   ├── C_UI_WeaponCard_ActiveBoost.uasset
│   │   │   ├── MF_UI_Absolute0to1.uasset
│   │   │   ├── MF_UI_CoolDownMask.uasset
│   │   │   ├── MF_UI_HalftoneMask.uasset
│   │   │   ├── MF_UI_RadialGrad0To1.uasset
│   │   │   ├── MF_UI_RoundedCornerBorder.uasset
│   │   │   ├── MF_UI_RoundedCornerSquare.uasset
│   │   │   ├── MF_UI_UVWidthByHeight.uasset
│   │   │   ├── MF_UI_VariableSharpener.uasset
│   │   │   ├── MI_UI_AbilityFailure_Shadow.uasset
│   │   │   ├── MI_UI_AccoladeBorder.uasset
│   │   │   ├── MI_UI_AccoladeBorder_Text.uasset
│   │   │   ├── MI_UI_AccoladeStar.uasset
│   │   │   ├── MI_UI_Accolade_DoubleElimination.uasset
│   │   │   ├── MI_UI_Accolade_QuadrupleElimination.uasset
│   │   │   ├── MI_UI_Accolade_Shadow.uasset
│   │   │   ├── MI_UI_Accolade_Streak10.uasset
│   │   │   ├── MI_UI_Accolade_Streak15.uasset
│   │   │   ├── MI_UI_Accolade_Streak20.uasset
│   │   │   ├── MI_UI_Accolade_Streak5.uasset
│   │   │   ├── MI_UI_Accolade_TripleElimination.uasset
│   │   │   ├── MI_UI_AmmoCounter_Pistol.uasset
│   │   │   ├── MI_UI_AmmoCounter_Pistol_Backing.uasset
│   │   │   ├── MI_UI_AmmoCounter_Pistol_Glow.uasset
│   │   │   ├── MI_UI_AmmoCounter_Rifle.uasset
│   │   │   ├── MI_UI_AmmoCounter_Rifle_Backing.uasset
│   │   │   ├── MI_UI_AmmoCounter_Rifle_Glow.uasset
│   │   │   ├── MI_UI_AmmoCounter_Shotgun.uasset
│   │   │   ├── MI_UI_AmmoCounter_Shotgun_Backing.uasset
│   │   │   ├── MI_UI_AmmoCounter_Shotgun_Glow.uasset
│   │   │   ├── MI_UI_Base_Icon_ElimFeed.uasset
│   │   │   ├── MI_UI_Base_WeaponCard.uasset
│   │   │   ├── MI_UI_Base_WeaponCard_CoolDown.uasset
│   │   │   ├── MI_UI_Base_WeaponCard_TouchButton.uasset
│   │   │   ├── MI_UI_ControlRing_Base.uasset
│   │   │   ├── MI_UI_ControlRing_Burst.uasset
│   │   │   ├── MI_UI_ControlRing_CenterFill.uasset
│   │   │   ├── MI_UI_ControlRing_Glow.uasset
│   │   │   ├── MI_UI_ControlRing_LetterColor.uasset
│   │   │   ├── MI_UI_ControlRing_Shadow.uasset
│   │   │   ├── MI_UI_ControlRing_SoftGlow.uasset
│   │   │   ├── MI_UI_Dash_Backing.uasset
│   │   │   ├── MI_UI_Dash_BarFill.uasset
│   │   │   ├── MI_UI_Dash_BarGlow.uasset
│   │   │   ├── MI_UI_Dash_BarGlow_Boost.uasset
│   │   │   ├── MI_UI_Dash_OuterRing.uasset
│   │   │   ├── MI_UI_ElimFeed_NameColor.uasset
│   │   │   ├── MI_UI_ElimFeed_NameGlow.uasset
│   │   │   ├── MI_UI_ElimFeed_Shadow.uasset
│   │   │   ├── MI_UI_HealthBar_BarBorder.uasset
│   │   │   ├── MI_UI_HealthBar_Fill.uasset
│   │   │   ├── MI_UI_HealthBar_Glow.uasset
│   │   │   ├── MI_UI_HealthBar_NumberBorder.uasset
│   │   │   ├── MI_UI_HealthBar_Shadow.uasset
│   │   │   ├── MI_UI_Icon_Ammo_Card.uasset
│   │   │   ├── MI_UI_Icon_Ammo_Counter.uasset
│   │   │   ├── MI_UI_Icon_ElimFeed_Grenade.uasset
│   │   │   ├── MI_UI_Icon_ElimFeed_Melee.uasset
│   │   │   ├── MI_UI_Icon_ElimFeed_Pistol.uasset
│   │   │   ├── MI_UI_Icon_ElimFeed_Rifle.uasset
│   │   │   ├── MI_UI_Icon_ElimFeed_SelfElimination.uasset
│   │   │   ├── MI_UI_Icon_ElimFeed_Shotgun.uasset
│   │   │   ├── MI_UI_QuickBar_AmmoCounterBorder.uasset
│   │   │   ├── MI_UI_QuickBar_Border.uasset
│   │   │   ├── MI_UI_QuickBar_Border_Square.uasset
│   │   │   ├── MI_UI_RespawnTimer.uasset
│   │   │   ├── MI_UI_RespawnTimer_Glow.uasset
│   │   │   ├── MI_UI_RespawnTimer_Shadow.uasset
│   │   │   ├── MI_UI_RespawnTimer_SoftGlow.uasset
│   │   │   ├── MI_UI_RespawnTimer_Text.uasset
│   │   │   ├── MI_UI_Reticles_BaseInstance.uasset
│   │   │   ├── MI_UI_Reticles_CrossHair_Pistol.uasset
│   │   │   ├── MI_UI_Reticles_CrossHair_Rifle.uasset
│   │   │   ├── MI_UI_Reticles_CrossHair_Shotgun.uasset
│   │   │   ├── MI_UI_Reticles_EliminationMarker.uasset
│   │   │   ├── MI_UI_Reticles_HitMarker.uasset
│   │   │   ├── MI_UI_Reticles_HitMarkerConfirmation.uasset
│   │   │   ├── MI_UI_Reticles_HitMarkerConfirmation_Critical.uasset
│   │   │   ├── MI_UI_Reticles_HitMarker_Single.uasset
│   │   │   ├── MI_UI_Reticles_Outer_Pistol.uasset
│   │   │   ├── MI_UI_Reticles_Outer_Rifle.uasset
│   │   │   ├── MI_UI_Reticles_Outer_Shotgun.uasset
│   │   │   ├── MI_UI_Reticles_TargetDot_FourSpread.uasset
│   │   │   ├── MI_UI_Reticles_TargetDot_Single.uasset
│   │   │   ├── MI_UI_TeamScore_Backing.uasset
│   │   │   ├── MI_UI_TeamScore_BarFill.uasset
│   │   │   ├── MI_UI_TeamScore_BarFill_Enemy.uasset
│   │   │   ├── MI_UI_TeamScore_BarFill_Glow.uasset
│   │   │   ├── MI_UI_TeamScore_BarFill_Glow_Enemy.uasset
│   │   │   ├── MI_UI_TeamScore_BarFill_NumberGlow.uasset
│   │   │   ├── MI_UI_TeamScore_BarFill_NumberGlow_Enemy.uasset
│   │   │   ├── MI_UI_TeamScore_Shadow.uasset
│   │   │   ├── MI_UI_TouchCard_ADS.uasset
│   │   │   ├── MI_UI_TouchCard_Crouch.uasset
│   │   │   ├── MI_UI_TouchCard_Dash.uasset
│   │   │   ├── MI_UI_TouchCard_Emote.uasset
│   │   │   ├── MI_UI_TouchCard_Jump.uasset
│   │   │   ├── MI_UI_TouchCard_Melee.uasset
│   │   │   ├── MI_UI_WeaponCard_CoolDown_Grenade.uasset
│   │   │   ├── MI_UI_WeaponCard_Glow.uasset
│   │   │   ├── MI_UI_WeaponCard_Glow_Boost.uasset
│   │   │   ├── MI_UI_WeaponCard_Glow_CoolDown.uasset
│   │   │   ├── MI_UI_WeaponCard_Glow_TouchButton.uasset
│   │   │   ├── MI_UI_WeaponCard_Glow_TouchButton_CoolDown.uasset
│   │   │   ├── M_UI_Base_AmmoCounter.uasset
│   │   │   ├── M_UI_Base_Control.uasset
│   │   │   ├── M_UI_Base_HealthBar.uasset
│   │   │   ├── M_UI_Base_ReticleBuilder.uasset
│   │   │   ├── M_UI_Base_SimpleIcon.uasset
│   │   │   ├── M_UI_Base_WeaponCard.uasset
│   │   │   ├── T_UI_Effects_TriHalftone_Mask.uasset
│   │   │   ├── T_UI_Icon_Abilities_Dash.uasset
│   │   │   ├── T_UI_Icon_Accolade_DoubleElimination.uasset
│   │   │   ├── T_UI_Icon_Accolade_QuadrupleElimination.uasset
│   │   │   ├── T_UI_Icon_Accolade_Streak10.uasset
│   │   │   ├── T_UI_Icon_Accolade_Streak15.uasset
│   │   │   ├── T_UI_Icon_Accolade_Streak20.uasset
│   │   │   ├── T_UI_Icon_Accolade_Streak5.uasset
│   │   │   ├── T_UI_Icon_Accolade_TripleElimination.uasset
│   │   │   ├── T_UI_Icon_MobileOnly_ADS.uasset
│   │   │   ├── T_UI_Icon_MobileOnly_Crouch.uasset
│   │   │   ├── T_UI_Icon_MobileOnly_Dash.uasset
│   │   │   ├── T_UI_Icon_MobileOnly_Emote.uasset
│   │   │   ├── T_UI_Icon_MobileOnly_Jump.uasset
│   │   │   ├── T_UI_Icon_OtherWeapons_Grenade.uasset
│   │   │   ├── T_UI_Icon_OtherWeapons_Melee.uasset
│   │   │   ├── T_UI_Icon_OtherWeapons_SelfElimination.uasset
│   │   │   ├── T_UI_Icon_RangedWeapons_Pistol.uasset
│   │   │   ├── T_UI_Icon_RangedWeapons_Pistol_Ammo.uasset
│   │   │   ├── T_UI_Icon_RangedWeapons_Rifle.uasset
│   │   │   ├── T_UI_Icon_RangedWeapons_Rifle_Ammo.uasset
│   │   │   ├── T_UI_Icon_RangedWeapons_Shotgun.uasset
│   │   │   └── T_UI_Icon_RangedWeapons_Shotgun_Ammo.uasset
│   │   ├── Textures/
│   │   │   └── T_Crosshair_Dot.uasset
│   │   ├── W_ActionTouchButton.uasset
│   │   ├── W_ActionTouchButton_MobileOnly.uasset
│   │   ├── W_DefaultHUDLayout.uasset
│   │   ├── W_Healthbar.uasset
│   │   ├── W_LyraGameMenu.uasset
│   │   ├── W_OnScreenJoystick_Left.uasset
│   │   ├── W_OnScreenJoystick_Right.uasset
│   │   ├── W_OpenMenuTouchButton.uasset
│   │   ├── W_TouchRegion_Base.uasset
│   │   └── W_TouchRegion_Right.uasset
│   ├── Indicators/
│   │   ├── Message_NameplateInfo.uasset
│   │   ├── Message_NameplateRequest.uasset
│   │   ├── NameplateManagerComponent.uasset
│   │   ├── NameplateSource.uasset
│   │   └── W_Nameplate.uasset
│   ├── Menu/
│   │   ├── Art/
│   │   │   ├── CA_UI_Frontend.uasset
│   │   │   ├── C_UI_LoadingScreen.uasset
│   │   │   ├── C_UI_Logo.uasset
│   │   │   ├── C_UI_MapTile.uasset
│   │   │   ├── C_UI_MenuBorder.uasset
│   │   │   ├── C_UI_MenuButton_Base.uasset
│   │   │   ├── C_UI_ScrollBar.uasset
│   │   │   ├── C_UI_TeamLogos.uasset
│   │   │   ├── C_UI_TeamStripe.uasset
│   │   │   ├── C_UI_TileArt.uasset
│   │   │   ├── MI_UI_BrowserMessageBorder.uasset
│   │   │   ├── MI_UI_ButtonTileArt.uasset
│   │   │   ├── MI_UI_CountdownActivatingText.uasset
│   │   │   ├── MI_UI_CountdownGlow.uasset
│   │   │   ├── MI_UI_CreditsLogo.uasset
│   │   │   ├── MI_UI_DefaultLoadingScreen.uasset
│   │   │   ├── MI_UI_ExperienceButton_Base.uasset
│   │   │   ├── MI_UI_ExperienceButton_Base_Ring.uasset
│   │   │   ├── MI_UI_ExperienceTileArt.uasset
│   │   │   ├── MI_UI_FontColor_BoundActionButton.uasset
│   │   │   ├── MI_UI_FontColor_BoundActionButton_DebugOnly.uasset
│   │   │   ├── MI_UI_FontColor_CountDownNumbers.uasset
│   │   │   ├── MI_UI_FontColor_MenuButton.uasset
│   │   │   ├── MI_UI_FontColor_SessionButton_High.uasset
│   │   │   ├── MI_UI_FontColor_SessionButton_High_Mid.uasset
│   │   │   ├── MI_UI_FontColor_SessionButton_White.uasset
│   │   │   ├── MI_UI_FontColor_SettingsEntry.uasset
│   │   │   ├── MI_UI_FontColor_SettingsEnum.uasset
│   │   │   ├── MI_UI_FontColor_SettingsValue.uasset
│   │   │   ├── MI_UI_FontColor_StartScreen.uasset
│   │   │   ├── MI_UI_FontColor_TabButton.uasset
│   │   │   ├── MI_UI_FontColor_TileButton.uasset
│   │   │   ├── MI_UI_LoadingSoftShadow.uasset
│   │   │   ├── MI_UI_LoadingSymbol.uasset
│   │   │   ├── MI_UI_LoadingSymbol_Sine.uasset
│   │   │   ├── MI_UI_LoadingSymbol_Sine_DefaultLoadingScreen.uasset
│   │   │   ├── MI_UI_LoadingSymbol_Sine_Modal.uasset
│   │   │   ├── MI_UI_LoadingTextColor.uasset
│   │   │   ├── MI_UI_Logo.uasset
│   │   │   ├── MI_UI_Logo_DefaultLoadingScreen.uasset
│   │   │   ├── MI_UI_Logo_Glow.uasset
│   │   │   ├── MI_UI_Logo_Glow_DefaultLoadingScreen.uasset
│   │   │   ├── MI_UI_Logo_SoftGlow.uasset
│   │   │   ├── MI_UI_Logo_SoftGlow_DefaultLoadingScreen.uasset
│   │   │   ├── MI_UI_Logo_SoftGlow_Loading.uasset
│   │   │   ├── MI_UI_Logo_SoftShadow.uasset
│   │   │   ├── MI_UI_MapTitleBorder.uasset
│   │   │   ├── MI_UI_MenuBorder_AdditiveBoost.uasset
│   │   │   ├── MI_UI_MenuBorder_AdditiveBoost_Vertical.uasset
│   │   │   ├── MI_UI_MenuBorder_Ring.uasset
│   │   │   ├── MI_UI_MenuBorder_SoftDarkBlue.uasset
│   │   │   ├── MI_UI_MenuBorder_SoftDarkBlue_HighOpacity.uasset
│   │   │   ├── MI_UI_MenuBorder_SoftDarkBlue_Opaque.uasset
│   │   │   ├── MI_UI_MenuButton_Base.uasset
│   │   │   ├── MI_UI_MenuButton_Base_Glow.uasset
│   │   │   ├── MI_UI_MenuButton_Base_Ring.uasset
│   │   │   ├── MI_UI_MenuButton_SoftShadow.uasset
│   │   │   ├── MI_UI_SafezoneAdjustors.uasset
│   │   │   ├── MI_UI_ScoreBoard_PingBorder.uasset
│   │   │   ├── MI_UI_ScoreBoard_ScoreFill_Blue.uasset
│   │   │   ├── MI_UI_ScoreBoard_ScoreFill_Red.uasset
│   │   │   ├── MI_UI_ScoreBoard_ScoreHeader_Blue.uasset
│   │   │   ├── MI_UI_ScoreBoard_ScoreHeader_Red.uasset
│   │   │   ├── MI_UI_Scoreboard_BackgroundDim.uasset
│   │   │   ├── MI_UI_Scoreboard_Glow_Blue.uasset
│   │   │   ├── MI_UI_Scoreboard_Glow_Red.uasset
│   │   │   ├── MI_UI_ScrollBar_Base.uasset
│   │   │   ├── MI_UI_ScrollBar_Hover.uasset
│   │   │   ├── MI_UI_ScrollBar_Shadow_Bottom.uasset
│   │   │   ├── MI_UI_ScrollBar_Shadow_Left.uasset
│   │   │   ├── MI_UI_ScrollBar_Shadow_Right.uasset
│   │   │   ├── MI_UI_ScrollBar_Shadow_Top.uasset
│   │   │   ├── MI_UI_ScrollBar_Track.uasset
│   │   │   ├── MI_UI_SessionButton_Base.uasset
│   │   │   ├── MI_UI_SessionButton_EmptyEntry.uasset
│   │   │   ├── MI_UI_SessionButton_PingBorder.uasset
│   │   │   ├── MI_UI_SessionButton_SettingBorder.uasset
│   │   │   ├── MI_UI_SettingsRotator.uasset
│   │   │   ├── MI_UI_SettingsRotator_SizeB.uasset
│   │   │   ├── MI_UI_SettingsRotator_SizeC.uasset
│   │   │   ├── MI_UI_SettingsRotator_SizeD.uasset
│   │   │   ├── MI_UI_Settings_ArrowIcon.uasset
│   │   │   ├── MI_UI_Settings_BodyBorder.uasset
│   │   │   ├── MI_UI_Settings_DiamondIcon.uasset
│   │   │   ├── MI_UI_Settings_EntryBorder.uasset
│   │   │   ├── MI_UI_Settings_EnumBorder.uasset
│   │   │   ├── MI_UI_Settings_Menu.uasset
│   │   │   ├── MI_UI_Settings_OptionBorder.uasset
│   │   │   ├── MI_UI_Settings_ProgressBar.uasset
│   │   │   ├── MI_UI_Settings_ResetArrow.uasset
│   │   │   ├── MI_UI_Settings_TabHeader.uasset
│   │   │   ├── MI_UI_Settings_XMarkIcon.uasset
│   │   │   ├── MI_UI_SimpleMenuButton_Base.uasset
│   │   │   ├── MI_UI_SimpleMenuButton_DebugOnly.uasset
│   │   │   ├── MI_UI_SimpleSpinner.uasset
│   │   │   ├── MI_UI_SimpleSpinner_Countdown.uasset
│   │   │   ├── MI_UI_SimpleSpinner_HighContrast.uasset
│   │   │   ├── MI_UI_StartScreen_Base.uasset
│   │   │   ├── MI_UI_TabButton_Base.uasset
│   │   │   ├── MI_UI_Teams_BackgroundDim.uasset
│   │   │   ├── MI_UI_Teams_Base.uasset
│   │   │   ├── MI_UI_Teams_BlueTeam.uasset
│   │   │   ├── MI_UI_Teams_BlueTeam_Glow.uasset
│   │   │   ├── MI_UI_Teams_BlueTeam_HealthBar.uasset
│   │   │   ├── MI_UI_Teams_BlueTeam_Mini.uasset
│   │   │   ├── MI_UI_Teams_BlueTeam_ScoreCounter.uasset
│   │   │   ├── MI_UI_Teams_HealthBarBacking.uasset
│   │   │   ├── MI_UI_Teams_LogoBorder.uasset
│   │   │   ├── MI_UI_Teams_NameplateGlow.uasset
│   │   │   ├── MI_UI_Teams_NameplateShadow.uasset
│   │   │   ├── MI_UI_Teams_RedTeam.uasset
│   │   │   ├── MI_UI_Teams_RedTeam_Glow.uasset
│   │   │   ├── MI_UI_Teams_RedTeam_HealthBar.uasset
│   │   │   ├── MI_UI_Teams_RedTeam_Mini.uasset
│   │   │   ├── MI_UI_Teams_RedTeam_ScoreCounter.uasset
│   │   │   ├── MI_UI_Teams_Ring.uasset
│   │   │   ├── MI_UI_Teams_SoftGlow_Blue.uasset
│   │   │   ├── MI_UI_Teams_SoftGlow_Red.uasset
│   │   │   ├── MI_UI_Teams_Stripe_Blue.uasset
│   │   │   ├── MI_UI_Teams_Stripe_Red.uasset
│   │   │   ├── MI_UI_Teams_TextBorder_Blue.uasset
│   │   │   ├── MI_UI_Teams_TextBorder_Red.uasset
│   │   │   ├── MI_UI_Teams_Text_Blue.uasset
│   │   │   ├── MI_UI_Teams_Text_Red.uasset
│   │   │   ├── MI_UI_TileButton_Base.uasset
│   │   │   ├── MI_UI_TileButton_Base_Glow.uasset
│   │   │   ├── MI_UI_TileButton_Base_Ring.uasset
│   │   │   ├── MI_UI_UE5Logo_Large.uasset
│   │   │   ├── MI_UI_UE5Logo_Small.uasset
│   │   │   ├── MI_UI_WaitingMessageBorder.uasset
│   │   │   ├── MI_UI_WaitingTextColor.uasset
│   │   │   ├── MI_UI_WaitingTextShadow.uasset
│   │   │   ├── M_UI_Base_BordersAndButtons.uasset
│   │   │   ├── M_UI_Base_LoadingSymbol.uasset
│   │   │   ├── M_UI_Base_Logo.uasset
│   │   │   ├── M_UI_Base_SettingsRotator.uasset
│   │   │   ├── M_UI_Base_SimpleFontColor.uasset
│   │   │   ├── M_UI_Base_TeamLogo.uasset
│   │   │   ├── M_UI_Base_TileArt.uasset
│   │   │   ├── T_UI_Icon_SimpleArrow.uasset
│   │   │   ├── T_UI_Icon_SimpleDiamond.uasset
│   │   │   ├── T_UI_Icon_SimpleMenu.uasset
│   │   │   ├── T_UI_Icon_SimpleXMark.uasset
│   │   │   ├── T_UI_Icon_Team_Art.uasset
│   │   │   ├── T_UI_Icon_Team_Art_Glow.uasset
│   │   │   ├── T_UI_Icon_Team_Art_Mini.uasset
│   │   │   ├── T_UI_Icon_Team_Art_Small.uasset
│   │   │   ├── T_UI_Icon_Team_Engineering.uasset
│   │   │   ├── T_UI_Icon_Team_Engineering_Glow.uasset
│   │   │   ├── T_UI_Icon_Team_Engineering_Mini.uasset
│   │   │   ├── T_UI_Icon_Team_Engineering_Small.uasset
│   │   │   ├── T_UI_LyraLogo.uasset
│   │   │   ├── T_UI_LyraLogo_2K.uasset
│   │   │   ├── T_UI_MapTile_Convolution.uasset
│   │   │   ├── T_UI_MapTile_Expanse.uasset
│   │   │   ├── T_UI_MapTile_NoIcon.uasset
│   │   │   ├── T_UI_TileArt_Browse.uasset
│   │   │   ├── T_UI_TileArt_Host.uasset
│   │   │   ├── T_UI_TileArt_Quickplay.uasset
│   │   │   └── T_UI_UE5Logo.uasset
│   │   ├── Experiences/
│   │   │   ├── UIExperienceMacros.uasset
│   │   │   ├── W_ExperienceList.uasset
│   │   │   ├── W_ExperienceSelectionScreen.uasset
│   │   │   ├── W_ExperienceTile.uasset
│   │   │   ├── W_HostSessionScreen.uasset
│   │   │   ├── W_LyraSessionButton.uasset
│   │   │   ├── W_NonInteractiveSpinner.uasset
│   │   │   ├── W_SessionBrowserEntry.uasset
│   │   │   └── W_SessionBrowserScreen.uasset
│   │   ├── Replays/
│   │   │   ├── W_ReplayBrowserScreen.uasset
│   │   │   └── W_ReplayListEntry.uasset
│   │   ├── LyraScrollBox.uasset
│   │   ├── MI_UI_TitleMaterial.uasset
│   │   ├── W_BuildConfiguration.uasset
│   │   ├── W_LyraArrowButton.uasset
│   │   ├── W_LyraExperienceTileButton.uasset
│   │   ├── W_LyraFrontEnd.uasset
│   │   ├── W_LyraMenuButton.uasset
│   │   ├── W_LyraMenuButton_Modal.uasset
│   │   ├── W_LyraStartup.uasset
│   │   ├── W_LyraTileButton.uasset
│   │   ├── W_TouchCloseButton.uasset
│   │   ├── W_UserLoginButton.uasset
│   │   └── W_UserWatermark.uasset
│   ├── PerfStats/
│   │   ├── W_PerfStatContainer_FrontEnd.uasset
│   │   ├── W_PerfStatContainer_GraphOnly.uasset
│   │   ├── W_PerfStatContainer_TextOnly.uasset
│   │   ├── W_SingleGraphStat.uasset
│   │   └── W_SingleTextStat.uasset
│   ├── Settings/
│   │   ├── Editors/
│   │   │   ├── W_SettingEntryBackground.uasset
│   │   │   ├── W_SettingsListEntry_Action.uasset
│   │   │   ├── W_SettingsListEntry_Discrete.uasset
│   │   │   ├── W_SettingsListEntry_Header.uasset
│   │   │   ├── W_SettingsListEntry_KBMBinding.uasset
│   │   │   ├── W_SettingsListEntry_Missing.uasset
│   │   │   ├── W_SettingsListEntry_Scalar.uasset
│   │   │   ├── W_SettingsListEntry_SubCollection.uasset
│   │   │   └── W_SettingsRotator.uasset
│   │   ├── Extensions/
│   │   │   ├── ColorBlind/
│   │   │   │   ├── W_ColorBlindExtension.uasset
│   │   │   │   ├── ishihara_all.uasset
│   │   │   │   ├── ishihara_g1.uasset
│   │   │   │   ├── ishihara_g2.uasset
│   │   │   │   ├── ishihara_g3.uasset
│   │   │   │   ├── ishihara_protanoomaly.uasset
│   │   │   │   └── ishihara_protanopia.uasset
│   │   │   ├── Enum/
│   │   │   │   ├── W_EnumOptionDetailsEntry.uasset
│   │   │   │   └── W_EnumOptionExtension.uasset
│   │   │   └── Subtitle/
│   │   │       ├── GameSubtitleOptions.uasset
│   │   │       └── W_Settings_SubtitlePreview.uasset
│   │   ├── Media/
│   │   │   ├── BottomSettingsPip_T.uasset
│   │   │   ├── M_UI_SettingBorder_Base.uasset
│   │   │   ├── M_UI_SettingsBorder.uasset
│   │   │   ├── M_UI_SettingsPanelBG.uasset
│   │   │   ├── M_UI_SettingsPanelBG_Inst.uasset
│   │   │   ├── SettingsDefaultSlider_M.uasset
│   │   │   ├── SettingsRotator_M.uasset
│   │   │   └── Window_Arrow_16x.uasset
│   │   ├── Screens/
│   │   │   ├── Gamma/
│   │   │   │   ├── T_GammaTestImage.uasset
│   │   │   │   └── W_GammaEditor.uasset
│   │   │   ├── SafeZone/
│   │   │   │   ├── MI_UI_RadialGradient.uasset
│   │   │   │   ├── M_Ui_SafeZoneCornerArrow.uasset
│   │   │   │   └── W_SafeZoneEditor.uasset
│   │   │   ├── W_KeyAlreadyBoundWarning.uasset
│   │   │   └── W_PressAnyKey.uasset
│   │   ├── DT_SaveActions.uasset
│   │   ├── GameSettingRegistryVisuals.uasset
│   │   ├── SettingsDescriptionStyles.uasset
│   │   ├── SettingsDescriptionWarningStyes.uasset
│   │   ├── W_GameSettingsDetailView.uasset
│   │   ├── W_LyraSettingScreen.uasset
│   │   └── W_SettingsPanel.uasset
│   ├── B_CommonInputData.uasset
│   ├── B_LyraFrontendStateComponent.uasset
│   ├── B_LyraUIPolicy.uasset
│   ├── DT_UniversalActions.uasset
│   └── W_OverallUILayout.uasset
├── Weapons/
│   ├── Generic/
│   │   ├── Materials/
│   │   │   ├── MI_InternalShapeGlow2_NoPattern.uasset
│   │   │   ├── MI_InternalShapeGlow_MinorPattern.uasset
│   │   │   ├── MI_InternalShapeGlow_NoPattern.uasset
│   │   │   ├── M_Glass.uasset
│   │   │   ├── M_InternalShapeGlow.uasset
│   │   │   ├── M_Item.uasset
│   │   │   └── M_Weapon.uasset
│   │   └── Textures/
│   │       ├── T_NoiseSpeckle_01.uasset
│   │       ├── T_SoftNoise.uasset
│   │       ├── T_Weapon_AORM.uasset
│   │       ├── T_Weapon_D.uasset
│   │       ├── T_Weapon_Mask.uasset
│   │       ├── T_Weapon_N.uasset
│   │       └── T_vk1iehhc_4K_Roughness.uasset
│   ├── Grenade/
│   │   ├── Material/
│   │   │   └── MI_Weapon_Grenade.uasset
│   │   ├── Mesh/
│   │   │   └── SM_grenade.uasset
│   │   └── Textures/
│   │       ├── T_Grenade_AORM.uasset
│   │       ├── T_Grenade_D.uasset
│   │       ├── T_Grenade_Masks.uasset
│   │       └── T_Grenade_N.uasset
│   ├── Healthpack/
│   │   ├── Material/
│   │   │   ├── MI_Item_HealthPackFull.uasset
│   │   │   └── MI_Item_HealthPackPart.uasset
│   │   ├── Mesh/
│   │   │   ├── SM_healthpackFull.uasset
│   │   │   └── SM_healthpackPart.uasset
│   │   └── Texture/
│   │       ├── T_HealthPackFull_AORM.uasset
│   │       ├── T_HealthPackFull_D.uasset
│   │       ├── T_HealthPackFull_Mask.uasset
│   │       ├── T_HealthPackFull_N.uasset
│   │       ├── T_HealthPackPart_AORM.uasset
│   │       ├── T_HealthPackPart_D.uasset
│   │       ├── T_HealthPackPart_Mask.uasset
│   │       └── T_HealthPackPart_N.uasset
│   ├── Pistol/
│   │   ├── Animations/
│   │   │   ├── ABP_Weap_Pistol.uasset
│   │   │   ├── AM_MM_Pistol_DryFire.uasset
│   │   │   ├── AM_MM_Pistol_Equip.uasset
│   │   │   ├── AM_MM_Pistol_Fire.uasset
│   │   │   ├── AM_MM_Pistol_Reload.uasset
│   │   │   ├── AM_MM_Pistol_Reload_Emote_MW.uasset
│   │   │   ├── AM_Weap_Pistol_Fire.uasset
│   │   │   ├── AM_Weap_Pistol_Reload.uasset
│   │   │   ├── Weap_Pistol_Fire.uasset
│   │   │   └── Weap_Pistol_Reload.uasset
│   │   ├── Materials/
│   │   │   └── MI_Weapon_Pistol.uasset
│   │   ├── Mesh/
│   │   │   ├── SK_Pistol.uasset
│   │   │   ├── SK_Pistol_PhysicsAsset.uasset
│   │   │   ├── SK_Pistol_Skeleton.uasset
│   │   │   └── SM_Pistol.uasset
│   │   ├── Textures/
│   │   │   ├── T_Pistol_AORM.uasset
│   │   │   ├── T_Pistol_D.uasset
│   │   │   ├── T_Pistol_Masks.uasset
│   │   │   └── T_Pistol_N.uasset
│   │   ├── GA_Weapon_Reload_Pistol.uasset
│   │   └── GE_Damage_Pistol.uasset
│   ├── Rifle/
│   │   ├── Animations/
│   │   │   ├── ABP_Weap_Rifle.uasset
│   │   │   ├── AM_MM_Rifle_DryFire.uasset
│   │   │   ├── AM_MM_Rifle_Equip.uasset
│   │   │   ├── AM_MM_Rifle_Fire.uasset
│   │   │   ├── AM_MM_Rifle_Reload.uasset
│   │   │   ├── AM_MM_Rifle_Reload_Emote_MW.uasset
│   │   │   ├── AM_Weap_Rifle_Fire.uasset
│   │   │   ├── AM_Weap_Rifle_Reload.uasset
│   │   │   ├── Weap_Rifle_Fire.uasset
│   │   │   └── Weap_Rifle_Reload.uasset
│   │   ├── Materials/
│   │   │   └── MI_Weapon_Rifle.uasset
│   │   ├── Mesh/
│   │   │   ├── SK_Rifle.uasset
│   │   │   ├── SK_Rifle_PhysicsAsset.uasset
│   │   │   ├── SK_Rifle_Skeleton.uasset
│   │   │   └── SM_Rifle.uasset
│   │   ├── Sounds/
│   │   │   └── Rifle_Load01.uasset
│   │   └── Textures/
│   │       ├── T_Rifle_AORM.uasset
│   │       ├── T_Rifle_Combined_N.uasset
│   │       ├── T_Rifle_D.uasset
│   │       └── T_Rifle_Masks.uasset
│   ├── Shotgun/
│   │   ├── Animations/
│   │   │   ├── ABP_Weap_Shotgun.uasset
│   │   │   ├── AM_MM_Shotgun_Fire.uasset
│   │   │   ├── AM_MM_Shotgun_Reload.uasset
│   │   │   ├── AM_MM_Shotgun_Reload_Emote_MW.uasset
│   │   │   ├── AM_Weap_Shotgun_Fire.uasset
│   │   │   ├── AM_Weap_Shotgun_Reload.uasset
│   │   │   ├── Weap_Shotgun_Fire.uasset
│   │   │   └── Weap_Shotgun_Reload.uasset
│   │   ├── Materials/
│   │   │   └── MI_Weapon_Shotgun.uasset
│   │   ├── Mesh/
│   │   │   ├── SKM_Shotgun.uasset
│   │   │   ├── SK_Shotgun_PhysicsAsset.uasset
│   │   │   ├── SK_Shotgun_Skeleton.uasset
│   │   │   └── SM_Shotgun.uasset
│   │   └── Texture/
│   │       ├── T_Shotgun_AORM.uasset
│   │       ├── T_Shotgun_D.uasset
│   │       ├── T_Shotgun_Mask.uasset
│   │       └── T_Shotgun_N.uasset
│   ├── Spawnpad/
│   │   ├── Material/
│   │   │   ├── MI_InternalShapeGlow.uasset
│   │   │   ├── MI_InternalShapeGlow_Circle.uasset
│   │   │   ├── MI_InternalShapeGlow_Hex.uasset
│   │   │   ├── MI_InternalShapeGlow_Square.uasset
│   │   │   └── M_MetalChroma_LaunchPad.uasset
│   │   ├── Mesh/
│   │   │   ├── SM_launchpad_Hex.uasset
│   │   │   ├── SM_launchpad_Round.uasset
│   │   │   └── SM_launchpad_Square.uasset
│   │   └── Texture/
│   │       └── T_LaunchPadShape.uasset
│   ├── Tests/
│   │   ├── B_ShootingTarget.uasset
│   │   ├── GE_HugeHealthTarget.uasset
│   │   ├── ShootingTarget_AbilitySet.uasset
│   │   └── ShootingTarget_PawnData.uasset
│   ├── B_Weapon.uasset
│   ├── GA_Weapon_AutoReload.uasset
│   ├── GA_Weapon_Fire.uasset
│   └── GA_Weapon_ReloadMagazine.uasset
├── B_LyraGameInstance.uasset
├── B_LyraGameMode.uasset
├── DefaultGameData.uasset
└── DefaultGame_Label.uasset
```
### 插件目录蓝图结构
```
插件目录蓝图结构: Plugins/
├── AsyncMixin/
│   ├── Source/
│   │   ├── Private/
│   │   ├── Public/
├── CommonGame/
│   ├── Source/
│   │   ├── Private/
│   │   │   ├── Actions/
│   │   │   ├── Messaging/
│   │   ├── Public/
│   │   │   ├── Actions/
│   │   │   ├── Messaging/
├── CommonLoadingScreen/
│   ├── Resources/
│   ├── Source/
│   │   ├── CommonLoadingScreen/
│   │   │   ├── Private/
│   │   │   ├── Public/
│   │   └── CommonStartupLoadingScreen/
│   │       ├── Private/
├── CommonUser/
│   ├── Resources/
│   ├── Source/
│   │   └── CommonUser/
│   │       ├── Private/
│   │       ├── Public/
├── GameFeatures/
│   ├── ShooterCore/
│   │   ├── Config/
│   │   │   └── Tags/
│   │   ├── Content/
│   │   │   ├── Accolades/
│   │   │   │   ├── AccoladeDataRegistry.uasset
│   │   │   │   ├── B_AccoladeRelay.uasset
│   │   │   │   ├── B_ElimChainProcessor.uasset
│   │   │   │   ├── B_ElimStreakProcessor.uasset
│   │   │   │   ├── B_EliminationFeedRelay.uasset
│   │   │   │   ├── DT_BasicShooterAccolades.uasset
│   │   │   │   └── EAS_BasicShooterAcolades.uasset
│   │   │   ├── Blueprint/
│   │   │   │   ├── Macros/
│   │   │   │   │   └── BPML_VFXMacros.uasset
│   │   │   │   ├── B_AbilitySpawner.uasset
│   │   │   │   ├── B_AimAssistTargetTest.uasset
│   │   │   │   ├── B_ControlPointVolume.uasset
│   │   │   │   ├── B_GrantAbilityPad_ApplyGE.uasset
│   │   │   │   ├── B_GrantInventoryPad_WithMesh.uasset
│   │   │   │   ├── B_GrantInventory_Pad.uasset
│   │   │   │   ├── B_Launcher.uasset
│   │   │   │   ├── B_Launcher_Push.uasset
│   │   │   │   ├── B_Launcher_Up.uasset
│   │   │   │   ├── B_PerfTestActor.uasset
│   │   │   │   ├── B_PerfTestSpawner.uasset
│   │   │   │   ├── B_Teleport.uasset
│   │   │   │   ├── B_WeaponSpawner.uasset
│   │   │   │   └── SM_XFacingCylinderUnit1.uasset
│   │   │   ├── Bot/
│   │   │   │   ├── BT/
│   │   │   │   │   ├── BB_Lyra_Shooter_Bot.uasset
│   │   │   │   │   ├── BTDecorator_AllowedToFire.uasset
│   │   │   │   │   └── BT_Lyra_Shooter_Bot.uasset
│   │   │   │   ├── EQS/
│   │   │   │   │   ├── Context/
│   │   │   │   │   │   ├── EQS_Context_ControlPoints.uasset
│   │   │   │   │   │   ├── EQS_Context_TargetEnemy.uasset
│   │   │   │   │   │   └── EQS_Context_WeaponCheck.uasset
│   │   │   │   │   ├── Generator/
│   │   │   │   │   │   └── EQS_GetAllEnemy.uasset
│   │   │   │   │   ├── EQS_AIPerceptionEnemy.uasset
│   │   │   │   │   ├── EQS_FindControlPoint.uasset
│   │   │   │   │   ├── EQS_FindTarget.uasset
│   │   │   │   │   ├── EQS_FindWeapon.uasset
│   │   │   │   │   └── EQS_MoveAgainstEnnemy.uasset
│   │   │   │   ├── Services/
│   │   │   │   │   ├── BTS_CheckAmmo.uasset
│   │   │   │   │   ├── BTS_ReloadWeapon.uasset
│   │   │   │   │   ├── BTS_SetFocus.uasset
│   │   │   │   │   └── BTS_Shoot.uasset
│   │   │   │   ├── B_AI_Controller_LyraShooter.uasset
│   │   │   │   ├── B_AI_Controller_LyraShooter_Passive.uasset
│   │   │   │   ├── B_ShooterBotSpawner.uasset
│   │   │   │   ├── B_ShooterBotSpawner_Perf.uasset
│   │   │   │   └── EQS_Tester.uasset
│   │   │   ├── Camera/
│   │   │   │   ├── CM_ThirdPersonADS.uasset
│   │   │   │   └── ThirdPersonADSOffsetCurve.uasset
│   │   │   ├── ControlPoint/
│   │   │   │   ├── AI/
│   │   │   │   │   ├── BT_Lyra_Shooter_ControlPoint_Bot.uasset
│   │   │   │   │   ├── B_AI_Controller_LyraShooter_ControlPoint.uasset
│   │   │   │   │   └── B_ShooterBotSpawner_ControlPoint.uasset
│   │   │   │   ├── UI/
│   │   │   │   │   ├── W_CPScoreWidget.uasset
│   │   │   │   │   ├── W_ControlPointMarker.uasset
│   │   │   │   │   ├── W_ControlPointStatusWidget.uasset
│   │   │   │   │   ├── W_MatchScoreBoard_CP.uasset
│   │   │   │   │   ├── W_SB_PlayerRow_CP.uasset
│   │   │   │   │   └── W_ScoreBoard_HeaderRow_CP.uasset
│   │   │   │   ├── AbilitySet_ControlPoint.uasset
│   │   │   │   ├── B_ControlPointScoring.uasset
│   │   │   │   ├── B_MusicManagerComponent_ControlPoint.uasset
│   │   │   │   └── GA_ShowLeaderboard_CP.uasset
│   │   │   ├── Effects/
│   │   │   │   ├── Environmental/
│   │   │   │   │   ├── NS_CapturePoint.uasset
│   │   │   │   │   └── NS_CapturePointCounter.uasset
│   │   │   │   ├── Material/
│   │   │   │   │   ├── MI_WeaponSpawnerTimerIndicator.uasset
│   │   │   │   │   ├── MT_BlackBorder.uasset
│   │   │   │   │   ├── MT_CP_Bar.uasset
│   │   │   │   │   ├── MT_ControlPoint.uasset
│   │   │   │   │   ├── M_CP.uasset
│   │   │   │   │   ├── M_GrenadeRadius.uasset
│   │   │   │   │   ├── M_Launcher.uasset
│   │   │   │   │   ├── M_Launcher_Inst.uasset
│   │   │   │   │   ├── M_Launcher_Inst1.uasset
│   │   │   │   │   ├── M_Launcher_Inst1_2.uasset
│   │   │   │   │   ├── M_Launcher_Inst1_3.uasset
│   │   │   │   │   ├── M_Launcher_Inst1_4.uasset
│   │   │   │   │   ├── M_Launcher_Inst2.uasset
│   │   │   │   │   ├── M_Launcher_Inst3.uasset
│   │   │   │   │   ├── M_Launcher_Inst4.uasset
│   │   │   │   │   ├── M_Launcher_Inst5.uasset
│   │   │   │   │   ├── M_Launcher_Inst6.uasset
│   │   │   │   │   ├── M_PadColor.uasset
│   │   │   │   │   └── M_WeaponSpawnTimerIndicator.uasset
│   │   │   │   └── Mesh/
│   │   │   │       └── capturepoint_ring_corner.uasset
│   │   │   ├── Elimination/
│   │   │   │   ├── UI/
│   │   │   │   │   ├── W_MatchScoreBoard_Elimination.uasset
│   │   │   │   │   ├── W_SB_PlayerRow_TDM.uasset
│   │   │   │   │   ├── W_ScoreBoard_HeaderRow_EliminationMode.uasset
│   │   │   │   │   └── W_ScoreWidget_Elimination.uasset
│   │   │   │   ├── AbilitySet_Elimination.uasset
│   │   │   │   ├── B_MusicManagerComponent_Elimination.uasset
│   │   │   │   ├── B_TeamDeathMatchScoring.uasset
│   │   │   │   └── GA_ShowLeaderboard_TDM.uasset
│   │   │   ├── Experiences/
│   │   │   │   ├── Phases/
│   │   │   │   │   ├── GE_DamageImmunity_FromGameMode.uasset
│   │   │   │   │   ├── GE_PregameLobby.uasset
│   │   │   │   │   ├── Phase_Playing.uasset
│   │   │   │   │   ├── Phase_PostGame.uasset
│   │   │   │   │   └── Phase_Warmup.uasset
│   │   │   │   ├── B_LyraShooterGame_ControlPoints.uasset
│   │   │   │   ├── B_ShooterGame_Elimination.uasset
│   │   │   │   ├── B_ShooterGame_Perf.uasset
│   │   │   │   ├── LAS_ShooterGame_SharedInput.uasset
│   │   │   │   ├── LAS_ShooterGame_StandardComponents.uasset
│   │   │   │   └── LAS_ShooterGame_StandardHUD.uasset
│   │   │   ├── Game/
│   │   │   │   ├── Dash/
│   │   │   │   │   ├── GA_Hero_Dash.uasset
│   │   │   │   │   ├── W_DashCooldown.uasset
│   │   │   │   │   └── W_DashTouchButton.uasset
│   │   │   │   ├── Emote/
│   │   │   │   │   ├── GA_Emote.uasset
│   │   │   │   │   └── W_EmoteTouchButton.uasset
│   │   │   │   ├── Melee/
│   │   │   │   │   ├── GA_Melee.uasset
│   │   │   │   │   └── W_MeleeTouchButton.uasset
│   │   │   │   ├── Respawn/
│   │   │   │   │   ├── GA_AutoRespawn.uasset
│   │   │   │   │   ├── GA_SpawnEffect.uasset
│   │   │   │   │   ├── GE_SpawnIn.uasset
│   │   │   │   │   └── W_RespawnTimer.uasset
│   │   │   │   ├── AbilitySet_ShooterHero.uasset
│   │   │   │   ├── B_HandleShooterReplays.uasset
│   │   │   │   ├── B_Hero_ShooterMannequin.uasset
│   │   │   │   ├── B_QuickBarComponent.uasset
│   │   │   │   ├── B_ShooterGameScoring_Base.uasset
│   │   │   │   ├── B_TeamColor_DeviceProperty.uasset
│   │   │   │   ├── B_TeamSetup_TwoTeams.uasset
│   │   │   │   ├── B_TeamSpawningRules.uasset
│   │   │   │   ├── GA_QuickbarSlots.uasset
│   │   │   │   ├── HeroData_ShooterGame.uasset
│   │   │   │   └── TagRelationships_ShooterHero.uasset
│   │   │   ├── GameplayCues/
│   │   │   │   ├── GCNL_Dash.uasset
│   │   │   │   ├── GCNL_Death.uasset
│   │   │   │   ├── GCNL_Launcher_Activate.uasset
│   │   │   │   ├── GCNL_MatchDecided.uasset
│   │   │   │   ├── GCNL_Spawning.uasset
│   │   │   │   ├── GCNL_Teleporter_Activate.uasset
│   │   │   │   ├── GCNL_WaitingForPlayers.uasset
│   │   │   │   ├── GCN_InteractPickUp.uasset
│   │   │   │   ├── GCN_Weapon_Melee.uasset
│   │   │   │   ├── GCN_Weapon_MeleeImpact.uasset
│   │   │   │   ├── W_MatchDecided_Message.uasset
│   │   │   │   └── W_WaitingForPlayers_Message.uasset
│   │   │   ├── Input/
│   │   │   │   ├── Abilities/
│   │   │   │   │   ├── GAB_ShowWidget_WhenInputPressed.uasset
│   │   │   │   │   ├── GAB_ShowWidget_WhileInputHeld.uasset
│   │   │   │   │   ├── GA_ADS.uasset
│   │   │   │   │   ├── GA_DropWeapon.uasset
│   │   │   │   │   ├── GA_Grenade.uasset
│   │   │   │   │   └── Struct_UIMessaging.uasset
│   │   │   │   ├── Actions/
│   │   │   │   │   ├── IA_ADS.uasset
│   │   │   │   │   ├── IA_DropWeapon.uasset
│   │   │   │   │   ├── IA_Emote.uasset
│   │   │   │   │   ├── IA_Grenade.uasset
│   │   │   │   │   ├── IA_Melee.uasset
│   │   │   │   │   ├── IA_QuickSlot1.uasset
│   │   │   │   │   ├── IA_QuickSlot2.uasset
│   │   │   │   │   ├── IA_QuickSlot3.uasset
│   │   │   │   │   ├── IA_QuickSlot_CycleBackward.uasset
│   │   │   │   │   ├── IA_QuickSlot_CycleForward.uasset
│   │   │   │   │   ├── IA_ShowScoreboard.uasset
│   │   │   │   │   └── InputData_ShooterGame_AddOns.uasset
│   │   │   │   ├── Components/
│   │   │   │   │   └── B_AimAssistTargetManager.uasset
│   │   │   │   ├── Mappings/
│   │   │   │   │   ├── C_AimAssistTargetWeightCurve.uasset
│   │   │   │   │   ├── IMC_ADS_Speed.uasset
│   │   │   │   │   ├── IMC_ShooterGame.uasset
│   │   │   │   │   └── IMC_ShooterGame_KBM.uasset
│   │   │   │   ├── Settings/
│   │   │   │   │   └── GamepadAimSensitivity_Targeting.uasset
│   │   │   │   └── W_TouchRegion_Left.uasset
│   │   │   ├── Item/
│   │   │   │   ├── B_CubeHat.uasset
│   │   │   │   └── B_HealPickup.uasset
│   │   │   ├── Items/
│   │   │   │   ├── HealthPickup/
│   │   │   │   │   ├── GE_InstantHeal_Big.uasset
│   │   │   │   │   ├── GE_InstantHeal_Part.uasset
│   │   │   │   │   ├── GE_InstantHeal_Pickup.uasset
│   │   │   │   │   ├── WeaponPickupData_Heal_Big.uasset
│   │   │   │   │   └── WeaponPickupData_Heal_Small.uasset
│   │   │   │   ├── HealthPickup_Unused/
│   │   │   │   │   ├── AbilitySet_HealPickup.uasset
│   │   │   │   │   ├── GA_HealPickup.uasset
│   │   │   │   │   ├── ID_HealPickup.uasset
│   │   │   │   │   └── WID_HealPickup.uasset
│   │   │   │   └── PhysicsCube/
│   │   │   │       ├── B_PhysicsTest.uasset
│   │   │   │       ├── MI_PhysicsTest.uasset
│   │   │   │       └── SM_1M_Cube.uasset
│   │   │   ├── Maps/
│   │   │   │   ├── L_ShooterGym.umap
│   │   │   │   └── L_ShooterPerf.umap
│   │   │   ├── System/
│   │   │   │   ├── Audio/
│   │   │   │   │   └── WeaponAudioFunctions.uasset
│   │   │   │   └── Playlists/
│   │   │   │       ├── DA_ShooterGame_ShooterGym.uasset
│   │   │   │       └── DA_ShooterGame_ShooterPerf.uasset
│   │   │   ├── UserInterface/
│   │   │   │   ├── HUD/
│   │   │   │   │   ├── MI_UI_AmmoArc.uasset
│   │   │   │   │   ├── M_UI_RadialProgress_Ability.uasset
│   │   │   │   │   ├── W_AbilityFailureFeedback.uasset
│   │   │   │   │   ├── W_AbilityProgress.uasset
│   │   │   │   │   ├── W_DebugWeaponSpreadWidget.uasset
│   │   │   │   │   ├── W_QuickBar.uasset
│   │   │   │   │   ├── W_QuickBarSlot.uasset
│   │   │   │   │   ├── W_Reticle_AmmoBar.uasset
│   │   │   │   │   ├── W_Temp_ReticleDot.uasset
│   │   │   │   │   ├── W_WeaponAmmoAndName.uasset
│   │   │   │   │   └── W_WeaponReticleHost.uasset
│   │   │   │   ├── Notifications/
│   │   │   │   │   ├── Accolades/
│   │   │   │   │   │   ├── W_AccoladeHostWidget.uasset
│   │   │   │   │   │   └── W_AccoladeToast.uasset
│   │   │   │   │   └── EliminationFeed/
│   │   │   │   │       ├── ElimFeedEntry.uasset
│   │   │   │   │       ├── EliminationFeedMessage.uasset
│   │   │   │   │       ├── W_EliminationFeed.uasset
│   │   │   │   │       └── W_EliminationFeedEntryWidget.uasset
│   │   │   │   ├── Replays/
│   │   │   │   │   ├── W_Replay_PlayerSelector.uasset
│   │   │   │   │   ├── W_Replay_ScrubBar.uasset
│   │   │   │   │   └── W_ShooterReplayHUD.uasset
│   │   │   │   ├── ScoreBoard/
│   │   │   │   │   ├── PlayerRow/
│   │   │   │   │   │   ├── W_BasePlayerRow.uasset
│   │   │   │   │   │   ├── W_SB_PlayerIcon.uasset
│   │   │   │   │   │   ├── W_SB_PlayerName.uasset
│   │   │   │   │   │   └── W_SB_PlayerStat.uasset
│   │   │   │   │   ├── W_SB_PlayerList.uasset
│   │   │   │   │   ├── W_SB_TeamStat.uasset
│   │   │   │   │   └── W_UserConnectionStatus.uasset
│   │   │   │   ├── ITakesLyraPlayerState.uasset
│   │   │   │   ├── W_FireButton.uasset
│   │   │   │   ├── W_ShooterHUDLayout.uasset
│   │   │   │   └── W_ToggleADSTouchButton.uasset
│   │   │   ├── Weapon/
│   │   │   │   └── Grenade/
│   │   │   │       └── B_Grenade.uasset
│   │   │   ├── Weapons/
│   │   │   │   ├── Grenade/
│   │   │   │   │   ├── Curve_GrenadeDamage.uasset
│   │   │   │   │   ├── GCN_Grenade_Detonate.uasset
│   │   │   │   │   ├── GE_Damage_Grenade.uasset
│   │   │   │   │   ├── GE_Grenade_Cooldown.uasset
│   │   │   │   │   └── W_GrenadeCooldown.uasset
│   │   │   │   ├── NetShooter_PROTO/
│   │   │   │   │   ├── AbilitySet_ShooterNetShooter.uasset
│   │   │   │   │   ├── B_NetShooter.uasset
│   │   │   │   │   ├── B_WeaponInstance_NetShooter.uasset
│   │   │   │   │   ├── GA_WeaponNetShooter.uasset
│   │   │   │   │   ├── ID_NetShooter.uasset
│   │   │   │   │   ├── WID_NetShooter.uasset
│   │   │   │   │   └── WeaponPickupData_NetShooter.uasset
│   │   │   │   ├── Pistol/
│   │   │   │   │   ├── AbilitySet_ShooterPistol.uasset
│   │   │   │   │   ├── B_Pistol.uasset
│   │   │   │   │   ├── B_WeaponInstance_Pistol.uasset
│   │   │   │   │   ├── GA_Weapon_Fire_Pistol.uasset
│   │   │   │   │   ├── GCN_Weapon_Pistol_Fire.uasset
│   │   │   │   │   ├── ID_Pistol.uasset
│   │   │   │   │   ├── WID_Pistol.uasset
│   │   │   │   │   ├── W_AmmoCounter_Pistol.uasset
│   │   │   │   │   ├── W_Reticle_Pistol.uasset
│   │   │   │   │   └── WeaponPickupData_Pistol.uasset
│   │   │   │   ├── Rifle/
│   │   │   │   │   ├── AbilitySet_ShooterRifle.uasset
│   │   │   │   │   ├── B_Rifle.uasset
│   │   │   │   │   ├── B_WeaponInstance_Rifle.uasset
│   │   │   │   │   ├── GA_Weapon_Fire_Rifle_Auto.uasset
│   │   │   │   │   ├── GA_Weapon_Reload_Rifle.uasset
│   │   │   │   │   ├── GCN_Weapon_Rifle_Fire.uasset
│   │   │   │   │   ├── GE_Damage_RifleAuto.uasset
│   │   │   │   │   ├── ID_Rifle.uasset
│   │   │   │   │   ├── WID_Rifle.uasset
│   │   │   │   │   ├── W_AmmoCounter_Rifle.uasset
│   │   │   │   │   ├── W_Reticle_Rifle.uasset
│   │   │   │   │   └── WeaponPickupData_Rifle.uasset
│   │   │   │   ├── Shotgun/
│   │   │   │   │   ├── AbilitySet_ShooterShotgun.uasset
│   │   │   │   │   ├── B_Shotgun.uasset
│   │   │   │   │   ├── B_WeaponInstance_Shotgun.uasset
│   │   │   │   │   ├── GA_Weapon_Fire_Shotgun.uasset
│   │   │   │   │   ├── GA_Weapon_Reload_NetShooter.uasset
│   │   │   │   │   ├── GA_Weapon_Reload_Shotgun.uasset
│   │   │   │   │   ├── GCN_Weapon_Shotgun_Fire.uasset
│   │   │   │   │   ├── GE_Damage_Shotgun.uasset
│   │   │   │   │   ├── ID_Shotgun.uasset
│   │   │   │   │   ├── WID_Shotgun.uasset
│   │   │   │   │   ├── W_AmmoCounter_Shotgun.uasset
│   │   │   │   │   ├── W_Reticle_Shotgun.uasset
│   │   │   │   │   └── WeaponPickupData_Shotgun.uasset
│   │   │   │   ├── B_WeaponInstance_Base.uasset
│   │   │   │   ├── GE_Damage_Melee.uasset
│   │   │   │   └── WID_InstantHeal.uasset
│   │   │   └── ShooterCore.uasset
│   │   ├── Resources/
│   │   ├── Source/
│   │   │   └── ShooterCoreRuntime/
│   │   │       ├── Private/
│   │   │       │   ├── Accolades/
│   │   │       │   ├── Input/
│   │   │       │   ├── MessageProcessors/
│   │   │       ├── Public/
│   │   │       │   ├── Accolades/
│   │   │       │   ├── Input/
│   │   │       │   ├── MessageProcessors/
│   │   │       │   ├── Messages/
│   ├── ShooterExplorer/
│   │   ├── Config/
│   │   │   └── Tags/
│   │   ├── Content/
│   │   │   ├── AI/
│   │   │   │   ├── BehaviorTrees/
│   │   │   │   │   ├── BB/
│   │   │   │   │   │   └── BB_UseSmartObject.uasset
│   │   │   │   │   ├── EnvQueries/
│   │   │   │   │   │   └── EQS_FindSmartObjects_Emote.uasset
│   │   │   │   │   ├── Tasks/
│   │   │   │   │   │   ├── BTT_FindAndClaimSmartObject.uasset
│   │   │   │   │   │   ├── BTT_GotoSmartObject.uasset
│   │   │   │   │   │   ├── BTT_ReleaseSmartObject.uasset
│   │   │   │   │   │   ├── BTT_UseSmartObject.uasset
│   │   │   │   │   │   └── BTT_UseSmartObject_2.uasset
│   │   │   │   │   └── BT_UseSmartObject.uasset
│   │   │   │   ├── SmartObjects/
│   │   │   │   │   ├── CAS_Data/
│   │   │   │   │   │   ├── CASR_1Char_1Obj.uasset
│   │   │   │   │   │   ├── CASR_2Char_1Obj.uasset
│   │   │   │   │   │   ├── CAS_Bench.uasset
│   │   │   │   │   │   ├── L_CAS_Bench.uasset
│   │   │   │   │   │   └── L_CAS_Boombox.uasset
│   │   │   │   │   ├── Definition/
│   │   │   │   │   │   ├── L_SOD_Bench_01.uasset
│   │   │   │   │   │   ├── L_SOD_Bench_02.uasset
│   │   │   │   │   │   ├── L_SOD_BoomBox.uasset
│   │   │   │   │   │   ├── L_SOD_EmotePad.uasset
│   │   │   │   │   │   └── L_SOD_SimpleBench_01.uasset
│   │   │   │   │   ├── Parents/
│   │   │   │   │   │   └── BP_SOActor_Parent_Actor.uasset
│   │   │   │   │   ├── StateTrees/
│   │   │   │   │   │   ├── L_SOST_BoomBox.uasset
│   │   │   │   │   │   ├── L_SOST_Emote.uasset
│   │   │   │   │   │   ├── L_SOST_IntroLoopOutro.uasset
│   │   │   │   │   │   └── L_SOST_SimpleCAS.uasset
│   │   │   │   │   ├── BP_SOActor_Bench.uasset
│   │   │   │   │   ├── BP_SOActor_Bench_01.uasset
│   │   │   │   │   ├── BP_SOActor_Bench_02.uasset
│   │   │   │   │   ├── BP_SOActor_BoomBox.uasset
│   │   │   │   │   ├── BP_SOActor_EmoteDance_01.uasset
│   │   │   │   │   ├── BP_SOActor_EmotePad_01.uasset
│   │   │   │   │   └── BP_SOActor_SimpleCASBench_01.uasset
│   │   │   │   ├── StateTrees/
│   │   │   │   │   ├── Evaluators/
│   │   │   │   │   │   └── EV_PlayerPosition.uasset
│   │   │   │   │   ├── Tasks/
│   │   │   │   │   │   ├── STT_FindAndClaimSmartObject.uasset
│   │   │   │   │   │   ├── STT_Follow.uasset
│   │   │   │   │   │   ├── STT_GetPlayerLocation.uasset
│   │   │   │   │   │   ├── STT_GetSlotLocation.uasset
│   │   │   │   │   │   ├── STT_GoTo.uasset
│   │   │   │   │   │   ├── STT_GoToAndUseSmartObject.uasset
│   │   │   │   │   │   ├── STT_GoTo_SmartObject.uasset
│   │   │   │   │   │   ├── STT_PickRandomMontage.uasset
│   │   │   │   │   │   ├── STT_PlayAnimMontage.uasset
│   │   │   │   │   │   ├── STT_PlayAnimMontage1.uasset
│   │   │   │   │   │   ├── STT_PlayRandomAnimMontage.uasset
│   │   │   │   │   │   ├── STT_ReleaseSmartObject.uasset
│   │   │   │   │   │   ├── STT_SimpleDebugDelays.uasset
│   │   │   │   │   │   └── STT_UseSmartObject.uasset
│   │   │   │   │   └── Trees/
│   │   │   │   │       ├── L_STT_FollowPlayer.uasset
│   │   │   │   │       ├── L_STT_TestStates.uasset
│   │   │   │   │       └── L_STT_UseSmartObject.uasset
│   │   │   │   └── UnrealFestDemo/
│   │   │   │       ├── L_DemoBench.uasset
│   │   │   │       ├── L_DemoDefinition.uasset
│   │   │   │       ├── L_DemoTree.uasset
│   │   │   │       ├── L_UnrealFestDemoMap.umap
│   │   │   │       └── L_UnrealFestShowcaseMap.umap
│   │   │   ├── Blueprint/
│   │   │   │   ├── BP_BoomBox.uasset
│   │   │   │   ├── BP_GameplayEffectPad_SpawnFollowHero.uasset
│   │   │   │   ├── BP_PadToggle_DataLayer.uasset
│   │   │   │   ├── BP_PadToggle_SmartObject.uasset
│   │   │   │   ├── B_Hero_Explorer.uasset
│   │   │   │   ├── B_Pawn_Explorer.uasset
│   │   │   │   ├── B_SimpleFollowPawn.uasset
│   │   │   │   └── B_SimpleFollowPawn_Big.uasset
│   │   │   ├── Game/
│   │   │   │   └── HeroData_Explorer.uasset
│   │   │   ├── Input/
│   │   │   │   ├── Abilities/
│   │   │   │   │   ├── AbilitySet_InventoryTest.uasset
│   │   │   │   │   ├── GA_Interact.uasset
│   │   │   │   │   ├── GA_ToggleInventory.uasset
│   │   │   │   │   ├── GA_ToggleMap.uasset
│   │   │   │   │   └── GA_ToggleMarkerInWorld.uasset
│   │   │   │   ├── Actions/
│   │   │   │   │   ├── IA_Interact.uasset
│   │   │   │   │   ├── IA_ToggleInventory.uasset
│   │   │   │   │   ├── IA_ToggleMap.uasset
│   │   │   │   │   ├── IA_ToggleMarkerInWorld.uasset
│   │   │   │   │   └── InputData_InventoryTest.uasset
│   │   │   │   └── Mappings/
│   │   │   │       └── IMC_InventoryTest.uasset
│   │   │   ├── Interact/
│   │   │   │   ├── GA_Interaction_Collect.uasset
│   │   │   │   └── GA_Interaction_Sit.uasset
│   │   │   ├── Items/
│   │   │   │   ├── B_InteractableChair.uasset
│   │   │   │   ├── B_InteractableRock.uasset
│   │   │   │   ├── TestID_Rock.uasset
│   │   │   │   └── TestID_Tree.uasset
│   │   │   ├── Maps/
│   │   │   │   ├── L_InteractionTestMap.umap
│   │   │   │   ├── L_InventoryTestMap.umap
│   │   │   │   ├── L_SmartObjects_TestMap.umap
│   │   │   │   └── L_StateTree_FollowTest.umap
│   │   │   ├── Meshes/
│   │   │   │   ├── Gyms/
│   │   │   │   │   ├── SM_ModuleA_090.uasset
│   │   │   │   │   ├── SM_NavCube.uasset
│   │   │   │   │   ├── SM_Railing_010.uasset
│   │   │   │   │   └── SM_stairs_1.uasset
│   │   │   │   └── Prop/
│   │   │   │       └── Kit_bench_RR/
│   │   │   │           ├── Material/
│   │   │   │           │   ├── MI_ParkBench.uasset
│   │   │   │           │   ├── MI_PierBench.uasset
│   │   │   │           │   └── M_BenchMaterial_Base.uasset
│   │   │   │           ├── Texture/
│   │   │   │           │   ├── parkBench/
│   │   │   │           │   │   ├── T_parkBench_AOMRD.uasset
│   │   │   │           │   │   ├── T_parkBench_C.uasset
│   │   │   │           │   │   └── T_parkBench_N.uasset
│   │   │   │           │   ├── pierBench/
│   │   │   │           │   │   ├── T_pierBench_AOMRD.uasset
│   │   │   │           │   │   ├── T_pierBench_C.uasset
│   │   │   │           │   │   └── T_pierBench_N.uasset
│   │   │   │           │   ├── Albedo.uasset
│   │   │   │           │   ├── Normal.uasset
│   │   │   │           │   ├── T_bench_C.uasset
│   │   │   │           │   ├── T_bench_M.uasset
│   │   │   │           │   ├── T_bench_N.uasset
│   │   │   │           │   └── whiteMask.uasset
│   │   │   │           └── mesh/
│   │   │   │               ├── SM_park_bench_N01.uasset
│   │   │   │               └── SM_pier_bench_N01.uasset
│   │   │   ├── System/
│   │   │   │   ├── DataLayers/
│   │   │   │   │   ├── L_Ai_TestLayer.uasset
│   │   │   │   │   ├── L_Ai_TestLayer_BaseContent.uasset
│   │   │   │   │   └── L_Ai_TestLayer_DynamicContent.uasset
│   │   │   │   └── Experiences/
│   │   │   │       ├── B_TestInventoryExperience.uasset
│   │   │   │       └── LAS_InventoryTest.uasset
│   │   │   ├── UserInterface/
│   │   │   │   ├── ItemAcquiredToastEntry.uasset
│   │   │   │   ├── TextStyle-Large.uasset
│   │   │   │   ├── W_InteractionPrompt.uasset
│   │   │   │   ├── W_InventoryGrid.uasset
│   │   │   │   ├── W_InventoryScreen.uasset
│   │   │   │   ├── W_InventoryTile.uasset
│   │   │   │   ├── W_ItemAcquiredList.uasset
│   │   │   │   ├── W_ItemAcquiredToastRow.uasset
│   │   │   │   └── W_MapScreen.uasset
│   │   │   └── ShooterExplorer.uasset
│   ├── ShooterMaps/
│   │   ├── Content/
│   │   │   ├── GameplayCues/
│   │   │   │   └── GC_Collect_Effect.uasset
│   │   │   ├── Items/
│   │   │   │   └── Backgrounds/
│   │   │   │       └── ShooterGameLobbyBG.uasset
│   │   │   ├── LevelSequence/
│   │   │   │   ├── SEQ_LevelDesignShowcaseA.uasset
│   │   │   │   ├── SEQ_LobbyScreen.uasset
│   │   │   │   ├── SEQ_LobbyScreenAnim.uasset
│   │   │   │   ├── SEQ_LobbyScreenAnim_2.uasset
│   │   │   │   └── SEQ_ScreenCaptures.uasset
│   │   │   ├── Maps/
│   │   │   │   ├── DataLayers/
│   │   │   │   │   ├── L_Convolution_Blockout/
│   │   │   │   │   │   ├── Gameplay.uasset
│   │   │   │   │   │   └── Layout.uasset
│   │   │   │   │   ├── L_Expanse/
│   │   │   │   │   │   ├── ExtraSpawn.uasset
│   │   │   │   │   │   ├── Gameplay.uasset
│   │   │   │   │   │   ├── LayoutModelA.uasset
│   │   │   │   │   │   └── Lighting.uasset
│   │   │   │   │   ├── L_Expanse_Blockout/
│   │   │   │   │   │   ├── ExtraSpawn.uasset
│   │   │   │   │   │   ├── Gameplay.uasset
│   │   │   │   │   │   ├── LayoutModelA.uasset
│   │   │   │   │   │   └── Lighting.uasset
│   │   │   │   │   └── L_FiringRange_WP/
│   │   │   │   │       ├── SM_ModuleA.uasset
│   │   │   │   │       └── SM_Railing.uasset
│   │   │   │   ├── L_Convolution_Blockout.umap
│   │   │   │   ├── L_Convolution_Blockout_BuiltData.uasset
│   │   │   │   ├── L_Expanse.umap
│   │   │   │   ├── L_Expanse_Blockout.umap
│   │   │   │   ├── L_Expanse_Blockout_BuiltData.uasset
│   │   │   │   ├── L_Expanse_BuiltData.uasset
│   │   │   │   ├── L_FiringRange_WP.umap
│   │   │   │   └── L_ShooterFrontendBackground.umap
│   │   │   ├── Meshes/
│   │   │   │   ├── Convolution_Blockout/
│   │   │   │   │   ├── MeshA_17F7D1.uasset
│   │   │   │   │   ├── MeshA_1EAE54.uasset
│   │   │   │   │   ├── MeshA_236ED6.uasset
│   │   │   │   │   ├── MeshA_26DCF6.uasset
│   │   │   │   │   ├── MeshA_2B2BFB.uasset
│   │   │   │   │   ├── MeshA_42F814.uasset
│   │   │   │   │   ├── MeshA_468CEA.uasset
│   │   │   │   │   ├── MeshA_4B79A0.uasset
│   │   │   │   │   ├── MeshA_4C179E.uasset
│   │   │   │   │   ├── MeshA_51F752.uasset
│   │   │   │   │   ├── MeshA_528CED.uasset
│   │   │   │   │   ├── MeshA_53AD96.uasset
│   │   │   │   │   ├── MeshA_53C696.uasset
│   │   │   │   │   ├── MeshA_544E7E.uasset
│   │   │   │   │   ├── MeshA_5E0D13.uasset
│   │   │   │   │   ├── MeshA_6A6823.uasset
│   │   │   │   │   ├── MeshA_761D63.uasset
│   │   │   │   │   ├── MeshA_7C4B2B.uasset
│   │   │   │   │   ├── MeshA_7EB701.uasset
│   │   │   │   │   ├── MeshA_898E56.uasset
│   │   │   │   │   ├── MeshA_8FCD15.uasset
│   │   │   │   │   ├── MeshA_931D72.uasset
│   │   │   │   │   ├── MeshA_98BFC7.uasset
│   │   │   │   │   ├── MeshA_9CD4DD.uasset
│   │   │   │   │   ├── MeshA_9E0475.uasset
│   │   │   │   │   ├── MeshA_A0E610.uasset
│   │   │   │   │   ├── MeshA_AECA18.uasset
│   │   │   │   │   ├── MeshA_B85A18.uasset
│   │   │   │   │   ├── MeshA_C223AB.uasset
│   │   │   │   │   ├── MeshA_C5D66C.uasset
│   │   │   │   │   ├── MeshA_DDDFE0.uasset
│   │   │   │   │   ├── MeshA_DEF968.uasset
│   │   │   │   │   ├── MeshA_E09F18.uasset
│   │   │   │   │   ├── MeshA_EF0433.uasset
│   │   │   │   │   ├── MeshA_F323C3.uasset
│   │   │   │   │   ├── MeshA_FCCF96.uasset
│   │   │   │   │   ├── MeshA_FE219F.uasset
│   │   │   │   │   ├── MeshA__1155CE.uasset
│   │   │   │   │   ├── MeshA__2AE6C2.uasset
│   │   │   │   │   ├── MeshA__85DCA1.uasset
│   │   │   │   │   ├── MeshA__8CD6AB.uasset
│   │   │   │   │   ├── MeshA__C44FF5.uasset
│   │   │   │   │   ├── MeshA__D24D1A.uasset
│   │   │   │   │   ├── MeshA__DDFE44.uasset
│   │   │   │   │   ├── Mesh__08803D.uasset
│   │   │   │   │   ├── Mesh__0FA9DE.uasset
│   │   │   │   │   ├── Mesh__13072C.uasset
│   │   │   │   │   └── Mesh__150F7B.uasset
│   │   │   │   ├── Expanse/
│   │   │   │   │   ├── Mesh_A_2_00B92E.uasset
│   │   │   │   │   ├── Mesh_A_2_01308C.uasset
│   │   │   │   │   ├── Mesh_A_2_029CA5.uasset
│   │   │   │   │   ├── Mesh_A_2_02F4BA.uasset
│   │   │   │   │   ├── Mesh_A_2_03EDC5.uasset
│   │   │   │   │   ├── Mesh_A_2_040F71.uasset
│   │   │   │   │   ├── Mesh_A_2_05663B.uasset
│   │   │   │   │   ├── Mesh_A_2_09F2DC.uasset
│   │   │   │   │   ├── Mesh_A_2_09FF96.uasset
│   │   │   │   │   ├── Mesh_A_2_0B044F.uasset
│   │   │   │   │   ├── Mesh_A_2_0B5F2E.uasset
│   │   │   │   │   ├── Mesh_A_2_0B8D9F.uasset
│   │   │   │   │   ├── Mesh_A_2_0CE8EE.uasset
│   │   │   │   │   ├── Mesh_A_2_0DFFED.uasset
│   │   │   │   │   ├── Mesh_A_2_0E1544.uasset
│   │   │   │   │   ├── Mesh_A_2_0E6029.uasset
│   │   │   │   │   ├── Mesh_A_2_0ED3A2.uasset
│   │   │   │   │   ├── Mesh_A_2_0EE11E.uasset
│   │   │   │   │   ├── Mesh_A_2_0F7966.uasset
│   │   │   │   │   ├── Mesh_A_2_0F799A.uasset
│   │   │   │   │   ├── Mesh_A_2_1016B8.uasset
│   │   │   │   │   ├── Mesh_A_2_12F2EB.uasset
│   │   │   │   │   ├── Mesh_A_2_16ABA8.uasset
│   │   │   │   │   ├── Mesh_A_2_16E57E.uasset
│   │   │   │   │   ├── Mesh_A_2_18A49A.uasset
│   │   │   │   │   ├── Mesh_A_2_18CF0C.uasset
│   │   │   │   │   ├── Mesh_A_2_1A1A13.uasset
│   │   │   │   │   ├── Mesh_A_2_1A257F.uasset
│   │   │   │   │   ├── Mesh_A_2_1AC7F7.uasset
│   │   │   │   │   ├── Mesh_A_2_1D0ED8.uasset
│   │   │   │   │   ├── Mesh_A_2_1E58A3.uasset
│   │   │   │   │   ├── Mesh_A_2_1ECB09.uasset
│   │   │   │   │   ├── Mesh_A_2_1EE512.uasset
│   │   │   │   │   ├── Mesh_A_2_219EDE.uasset
│   │   │   │   │   ├── Mesh_A_2_24B46A.uasset
│   │   │   │   │   ├── Mesh_A_2_25BB70.uasset
│   │   │   │   │   ├── Mesh_A_2_25F7B2.uasset
│   │   │   │   │   ├── Mesh_A_2_282B43.uasset
│   │   │   │   │   ├── Mesh_A_2_2831B6.uasset
│   │   │   │   │   ├── Mesh_A_2_285D49.uasset
│   │   │   │   │   ├── Mesh_A_2_287C2A.uasset
│   │   │   │   │   ├── Mesh_A_2_28A2AC.uasset
│   │   │   │   │   ├── Mesh_A_2_28DD2C.uasset
│   │   │   │   │   ├── Mesh_A_2_291D42.uasset
│   │   │   │   │   ├── Mesh_A_2_29959A.uasset
│   │   │   │   │   ├── Mesh_A_2_2C0BF7.uasset
│   │   │   │   │   ├── Mesh_A_2_2C9F67.uasset
│   │   │   │   │   ├── Mesh_A_2_2CF3C6.uasset
│   │   │   │   │   ├── Mesh_A_2_2D6CAC.uasset
│   │   │   │   │   ├── Mesh_A_2_2E79BE.uasset
│   │   │   │   │   ├── Mesh_A_2_300D10.uasset
│   │   │   │   │   ├── Mesh_A_2_300F08.uasset
│   │   │   │   │   ├── Mesh_A_2_30182F.uasset
│   │   │   │   │   ├── Mesh_A_2_30E606.uasset
│   │   │   │   │   ├── Mesh_A_2_31388C.uasset
│   │   │   │   │   ├── Mesh_A_2_3204DD.uasset
│   │   │   │   │   ├── Mesh_A_2_36AA2A.uasset
│   │   │   │   │   ├── Mesh_A_2_381B24.uasset
│   │   │   │   │   ├── Mesh_A_2_3863E3.uasset
│   │   │   │   │   ├── Mesh_A_2_390AC5.uasset
│   │   │   │   │   ├── Mesh_A_2_390D5E.uasset
│   │   │   │   │   ├── Mesh_A_2_3A9D55.uasset
│   │   │   │   │   ├── Mesh_A_2_3B823E.uasset
│   │   │   │   │   ├── Mesh_A_2_3BFC77.uasset
│   │   │   │   │   ├── Mesh_A_2_3C3D27.uasset
│   │   │   │   │   ├── Mesh_A_2_3C64E8.uasset
│   │   │   │   │   ├── Mesh_A_2_3F085E.uasset
│   │   │   │   │   ├── Mesh_A_2_3F45BD.uasset
│   │   │   │   │   ├── Mesh_A_2_40429E.uasset
│   │   │   │   │   ├── Mesh_A_2_40987B.uasset
│   │   │   │   │   ├── Mesh_A_2_40D9C0.uasset
│   │   │   │   │   ├── Mesh_A_2_412B3D.uasset
│   │   │   │   │   ├── Mesh_A_2_42BFF1.uasset
│   │   │   │   │   ├── Mesh_A_2_435A42.uasset
│   │   │   │   │   ├── Mesh_A_2_440E92.uasset
│   │   │   │   │   ├── Mesh_A_2_44EBD6.uasset
│   │   │   │   │   ├── Mesh_A_2_46A34E.uasset
│   │   │   │   │   ├── Mesh_A_2_46ACC7.uasset
│   │   │   │   │   ├── Mesh_A_2_47675F.uasset
│   │   │   │   │   ├── Mesh_A_2_4791EE.uasset
│   │   │   │   │   ├── Mesh_A_2_47DB7C.uasset
│   │   │   │   │   ├── Mesh_A_2_48C7C1.uasset
│   │   │   │   │   ├── Mesh_A_2_4935AE.uasset
│   │   │   │   │   ├── Mesh_A_2_499DF9.uasset
│   │   │   │   │   ├── Mesh_A_2_4B5953.uasset
│   │   │   │   │   ├── Mesh_A_2_4B645A.uasset
│   │   │   │   │   ├── Mesh_A_2_4C3618.uasset
│   │   │   │   │   ├── Mesh_A_2_4DBC05.uasset
│   │   │   │   │   ├── Mesh_A_2_4E3294.uasset
│   │   │   │   │   ├── Mesh_A_2_4EA8AE.uasset
│   │   │   │   │   ├── Mesh_A_2_5028E3.uasset
│   │   │   │   │   ├── Mesh_A_2_515FD9.uasset
│   │   │   │   │   ├── Mesh_A_2_518C36.uasset
│   │   │   │   │   ├── Mesh_A_2_519C61.uasset
│   │   │   │   │   ├── Mesh_A_2_53E335.uasset
│   │   │   │   │   ├── Mesh_A_2_54094D.uasset
│   │   │   │   │   ├── Mesh_A_2_541EAC.uasset
│   │   │   │   │   ├── Mesh_A_2_544E9B.uasset
│   │   │   │   │   ├── Mesh_A_2_5461E1.uasset
│   │   │   │   │   ├── Mesh_A_2_548D6B.uasset
│   │   │   │   │   ├── Mesh_A_2_55A5A8.uasset
│   │   │   │   │   ├── Mesh_A_2_55CE0C.uasset
│   │   │   │   │   ├── Mesh_A_2_565B55.uasset
│   │   │   │   │   ├── Mesh_A_2_57D463.uasset
│   │   │   │   │   ├── Mesh_A_2_57FA6B.uasset
│   │   │   │   │   ├── Mesh_A_2_58DDD6.uasset
│   │   │   │   │   ├── Mesh_A_2_58DE09.uasset
│   │   │   │   │   ├── Mesh_A_2_5A3D39.uasset
│   │   │   │   │   ├── Mesh_A_2_5B464C.uasset
│   │   │   │   │   ├── Mesh_A_2_5B848E.uasset
│   │   │   │   │   ├── Mesh_A_2_5C9C02.uasset
│   │   │   │   │   ├── Mesh_A_2_5CC092.uasset
│   │   │   │   │   ├── Mesh_A_2_5D0865.uasset
│   │   │   │   │   ├── Mesh_A_2_5D2634.uasset
│   │   │   │   │   ├── Mesh_A_2_5E608F.uasset
│   │   │   │   │   ├── Mesh_A_2_5E7B8E.uasset
│   │   │   │   │   ├── Mesh_A_2_5FC924.uasset
│   │   │   │   │   ├── Mesh_A_2_602A69.uasset
│   │   │   │   │   ├── Mesh_A_2_6091BB.uasset
│   │   │   │   │   ├── Mesh_A_2_6150F1.uasset
│   │   │   │   │   ├── Mesh_A_2_6183EC.uasset
│   │   │   │   │   ├── Mesh_A_2_6263F1.uasset
│   │   │   │   │   ├── Mesh_A_2_626D04.uasset
│   │   │   │   │   ├── Mesh_A_2_62EDBC.uasset
│   │   │   │   │   ├── Mesh_A_2_6475C0.uasset
│   │   │   │   │   ├── Mesh_A_2_64C304.uasset
│   │   │   │   │   ├── Mesh_A_2_6533F5.uasset
│   │   │   │   │   ├── Mesh_A_2_668B4D.uasset
│   │   │   │   │   ├── Mesh_A_2_66C10A.uasset
│   │   │   │   │   ├── Mesh_A_2_66D8ED.uasset
│   │   │   │   │   ├── Mesh_A_2_674F70.uasset
│   │   │   │   │   ├── Mesh_A_2_6755C8.uasset
│   │   │   │   │   ├── Mesh_A_2_68F12D.uasset
│   │   │   │   │   ├── Mesh_A_2_6A12A8.uasset
│   │   │   │   │   ├── Mesh_A_2_6AA72B.uasset
│   │   │   │   │   ├── Mesh_A_2_6B0290.uasset
│   │   │   │   │   ├── Mesh_A_2_6C9462.uasset
│   │   │   │   │   ├── Mesh_A_2_6D8992.uasset
│   │   │   │   │   ├── Mesh_A_2_6DD0A5.uasset
│   │   │   │   │   ├── Mesh_A_2_6DDB71.uasset
│   │   │   │   │   ├── Mesh_A_2_6E5781.uasset
│   │   │   │   │   ├── Mesh_A_2_6E6D31.uasset
│   │   │   │   │   ├── Mesh_A_2_6EC5BE.uasset
│   │   │   │   │   ├── Mesh_A_2_71F876.uasset
│   │   │   │   │   ├── Mesh_A_2_7350B6.uasset
│   │   │   │   │   ├── Mesh_A_2_7378C0.uasset
│   │   │   │   │   ├── Mesh_A_2_7386B9.uasset
│   │   │   │   │   ├── Mesh_A_2_73BCB5.uasset
│   │   │   │   │   ├── Mesh_A_2_75A15E.uasset
│   │   │   │   │   ├── Mesh_A_2_75A17D.uasset
│   │   │   │   │   ├── Mesh_A_2_75A517.uasset
│   │   │   │   │   ├── Mesh_A_2_75CE0F.uasset
│   │   │   │   │   ├── Mesh_A_2_76B214.uasset
│   │   │   │   │   ├── Mesh_A_2_7715CE.uasset
│   │   │   │   │   ├── Mesh_A_2_778EF5.uasset
│   │   │   │   │   ├── Mesh_A_2_7869F3.uasset
│   │   │   │   │   ├── Mesh_A_2_7AAD47.uasset
│   │   │   │   │   ├── Mesh_A_2_7AC870.uasset
│   │   │   │   │   ├── Mesh_A_2_7B0F50.uasset
│   │   │   │   │   ├── Mesh_A_2_7B75E5.uasset
│   │   │   │   │   ├── Mesh_A_2_7C67E5.uasset
│   │   │   │   │   ├── Mesh_A_2_7D7382.uasset
│   │   │   │   │   ├── Mesh_A_2_7DA053.uasset
│   │   │   │   │   ├── Mesh_A_2_7E1711.uasset
│   │   │   │   │   ├── Mesh_A_2_7E1EFF.uasset
│   │   │   │   │   ├── Mesh_A_2_82557F.uasset
│   │   │   │   │   ├── Mesh_A_2_82FBB8.uasset
│   │   │   │   │   ├── Mesh_A_2_8352AC.uasset
│   │   │   │   │   ├── Mesh_A_2_863DA2.uasset
│   │   │   │   │   ├── Mesh_A_2_86B8FC.uasset
│   │   │   │   │   ├── Mesh_A_2_87783F.uasset
│   │   │   │   │   ├── Mesh_A_2_87DE16.uasset
│   │   │   │   │   ├── Mesh_A_2_88BCCF.uasset
│   │   │   │   │   ├── Mesh_A_2_8C8BA9.uasset
│   │   │   │   │   ├── Mesh_A_2_8F6760.uasset
│   │   │   │   │   ├── Mesh_A_2_8FB4DA.uasset
│   │   │   │   │   ├── Mesh_A_2_90A919.uasset
│   │   │   │   │   ├── Mesh_A_2_914EFE.uasset
│   │   │   │   │   ├── Mesh_A_2_93E969.uasset
│   │   │   │   │   ├── Mesh_A_2_94ED4E.uasset
│   │   │   │   │   ├── Mesh_A_2_967D3A.uasset
│   │   │   │   │   ├── Mesh_A_2_96C65A.uasset
│   │   │   │   │   ├── Mesh_A_2_9940C7.uasset
│   │   │   │   │   ├── Mesh_A_2_9964B7.uasset
│   │   │   │   │   ├── Mesh_A_2_99A201.uasset
│   │   │   │   │   ├── Mesh_A_2_99AF90.uasset
│   │   │   │   │   ├── Mesh_A_2_9AA6C7.uasset
│   │   │   │   │   ├── Mesh_A_2_9AA9E2.uasset
│   │   │   │   │   ├── Mesh_A_2_9ABC14.uasset
│   │   │   │   │   ├── Mesh_A_2_9AC198.uasset
│   │   │   │   │   ├── Mesh_A_2_9C7ACC.uasset
│   │   │   │   │   ├── Mesh_A_2_9C9CCA.uasset
│   │   │   │   │   ├── Mesh_A_2_9CD27F.uasset
│   │   │   │   │   ├── Mesh_A_2_9D2FA2.uasset
│   │   │   │   │   ├── Mesh_A_2_9E4FA3.uasset
│   │   │   │   │   ├── Mesh_A_2_A14CD1.uasset
│   │   │   │   │   ├── Mesh_A_2_A23660.uasset
│   │   │   │   │   ├── Mesh_A_2_A35EAD.uasset
│   │   │   │   │   ├── Mesh_A_2_A4C633.uasset
│   │   │   │   │   ├── Mesh_A_2_A522DA.uasset
│   │   │   │   │   ├── Mesh_A_2_A5F894.uasset
│   │   │   │   │   ├── Mesh_A_2_A66FC6.uasset
│   │   │   │   │   ├── Mesh_A_2_A6AC56.uasset
│   │   │   │   │   ├── Mesh_A_2_A760DC.uasset
│   │   │   │   │   ├── Mesh_A_2_A7D0BC.uasset
│   │   │   │   │   ├── Mesh_A_2_A82418.uasset
│   │   │   │   │   ├── Mesh_A_2_A83366.uasset
│   │   │   │   │   ├── Mesh_A_2_A95EFA.uasset
│   │   │   │   │   ├── Mesh_A_2_A98BBB.uasset
│   │   │   │   │   ├── Mesh_A_2_AAB9D4.uasset
│   │   │   │   │   ├── Mesh_A_2_AB2B07.uasset
│   │   │   │   │   ├── Mesh_A_2_ABF695.uasset
│   │   │   │   │   ├── Mesh_A_2_AC587F.uasset
│   │   │   │   │   ├── Mesh_A_2_AC8176.uasset
│   │   │   │   │   ├── Mesh_A_2_AD71D2.uasset
│   │   │   │   │   ├── Mesh_A_2_ADBD2B.uasset
│   │   │   │   │   ├── Mesh_A_2_AEB1AF.uasset
│   │   │   │   │   ├── Mesh_A_2_B0D608.uasset
│   │   │   │   │   ├── Mesh_A_2_B12CED.uasset
│   │   │   │   │   ├── Mesh_A_2_B241CA.uasset
│   │   │   │   │   ├── Mesh_A_2_B281E9.uasset
│   │   │   │   │   ├── Mesh_A_2_B329AD.uasset
│   │   │   │   │   ├── Mesh_A_2_B451E3.uasset
│   │   │   │   │   ├── Mesh_A_2_B50523.uasset
│   │   │   │   │   ├── Mesh_A_2_B5D1BF.uasset
│   │   │   │   │   ├── Mesh_A_2_B61E45.uasset
│   │   │   │   │   ├── Mesh_A_2_B6E976.uasset
│   │   │   │   │   ├── Mesh_A_2_B83448.uasset
│   │   │   │   │   ├── Mesh_A_2_B91289.uasset
│   │   │   │   │   ├── Mesh_A_2_B9133A.uasset
│   │   │   │   │   ├── Mesh_A_2_B91A4B.uasset
│   │   │   │   │   ├── Mesh_A_2_B93EB0.uasset
│   │   │   │   │   ├── Mesh_A_2_B94F97.uasset
│   │   │   │   │   ├── Mesh_A_2_B9732E.uasset
│   │   │   │   │   ├── Mesh_A_2_BB5906.uasset
│   │   │   │   │   ├── Mesh_A_2_BB85A6.uasset
│   │   │   │   │   ├── Mesh_A_2_BBA547.uasset
│   │   │   │   │   ├── Mesh_A_2_BBFF8B.uasset
│   │   │   │   │   ├── Mesh_A_2_BC948F.uasset
│   │   │   │   │   ├── Mesh_A_2_BCAFE2.uasset
│   │   │   │   │   ├── Mesh_A_2_BCC5ED.uasset
│   │   │   │   │   ├── Mesh_A_2_BF565B.uasset
│   │   │   │   │   ├── Mesh_A_2_C0925B.uasset
│   │   │   │   │   ├── Mesh_A_2_C0E44F.uasset
│   │   │   │   │   ├── Mesh_A_2_C1120D.uasset
│   │   │   │   │   ├── Mesh_A_2_C18AE2.uasset
│   │   │   │   │   ├── Mesh_A_2_C2CE57.uasset
│   │   │   │   │   ├── Mesh_A_2_C33CFB.uasset
│   │   │   │   │   ├── Mesh_A_2_C3977A.uasset
│   │   │   │   │   ├── Mesh_A_2_C3E13A.uasset
│   │   │   │   │   ├── Mesh_A_2_C5005E.uasset
│   │   │   │   │   ├── Mesh_A_2_C56B37.uasset
│   │   │   │   │   ├── Mesh_A_2_C5797F.uasset
│   │   │   │   │   ├── Mesh_A_2_C6565E.uasset
│   │   │   │   │   ├── Mesh_A_2_C85750.uasset
│   │   │   │   │   ├── Mesh_A_2_C8AD57.uasset
│   │   │   │   │   ├── Mesh_A_2_C963E5.uasset
│   │   │   │   │   ├── Mesh_A_2_CA7E79.uasset
│   │   │   │   │   ├── Mesh_A_2_CB0246.uasset
│   │   │   │   │   ├── Mesh_A_2_CB38AF.uasset
│   │   │   │   │   ├── Mesh_A_2_CCF258.uasset
│   │   │   │   │   ├── Mesh_A_2_CD4CE4.uasset
│   │   │   │   │   ├── Mesh_A_2_CE3131.uasset
│   │   │   │   │   ├── Mesh_A_2_CE39D7.uasset
│   │   │   │   │   ├── Mesh_A_2_CE8BFF.uasset
│   │   │   │   │   ├── Mesh_A_2_CED0F7.uasset
│   │   │   │   │   ├── Mesh_A_2_CEE282.uasset
│   │   │   │   │   ├── Mesh_A_2_CF8333.uasset
│   │   │   │   │   ├── Mesh_A_2_CFB02B.uasset
│   │   │   │   │   ├── Mesh_A_2_D03773.uasset
│   │   │   │   │   ├── Mesh_A_2_D21E89.uasset
│   │   │   │   │   ├── Mesh_A_2_D26128.uasset
│   │   │   │   │   ├── Mesh_A_2_D29BDA.uasset
│   │   │   │   │   ├── Mesh_A_2_D59A41.uasset
│   │   │   │   │   ├── Mesh_A_2_D6817D.uasset
│   │   │   │   │   ├── Mesh_A_2_D6B253.uasset
│   │   │   │   │   ├── Mesh_A_2_D71D44.uasset
│   │   │   │   │   ├── Mesh_A_2_D725FD.uasset
│   │   │   │   │   ├── Mesh_A_2_D7370E.uasset
│   │   │   │   │   ├── Mesh_A_2_D83A6F.uasset
│   │   │   │   │   ├── Mesh_A_2_D8DF11.uasset
│   │   │   │   │   ├── Mesh_A_2_D8FC82.uasset
│   │   │   │   │   ├── Mesh_A_2_D99A95.uasset
│   │   │   │   │   ├── Mesh_A_2_DD2052.uasset
│   │   │   │   │   ├── Mesh_A_2_DEA3E4.uasset
│   │   │   │   │   ├── Mesh_A_2_DFA3EF.uasset
│   │   │   │   │   ├── Mesh_A_2_E0939C.uasset
│   │   │   │   │   ├── Mesh_A_2_E1239A.uasset
│   │   │   │   │   ├── Mesh_A_2_E222D0.uasset
│   │   │   │   │   ├── Mesh_A_2_E2B5F0.uasset
│   │   │   │   │   ├── Mesh_A_2_E4871D.uasset
│   │   │   │   │   ├── Mesh_A_2_E6A5E5.uasset
│   │   │   │   │   ├── Mesh_A_2_E7028D.uasset
│   │   │   │   │   ├── Mesh_A_2_E7FA38.uasset
│   │   │   │   │   ├── Mesh_A_2_E850CA.uasset
│   │   │   │   │   ├── Mesh_A_2_E97AAF.uasset
│   │   │   │   │   ├── Mesh_A_2_E97F0D.uasset
│   │   │   │   │   ├── Mesh_A_2_EA21AF.uasset
│   │   │   │   │   ├── Mesh_A_2_EA5470.uasset
│   │   │   │   │   ├── Mesh_A_2_EAE80F.uasset
│   │   │   │   │   ├── Mesh_A_2_EBAD85.uasset
│   │   │   │   │   ├── Mesh_A_2_ED9231.uasset
│   │   │   │   │   ├── Mesh_A_2_EE66FF.uasset
│   │   │   │   │   ├── Mesh_A_2_EE7365.uasset
│   │   │   │   │   ├── Mesh_A_2_EF0EBF.uasset
│   │   │   │   │   ├── Mesh_A_2_F00149.uasset
│   │   │   │   │   ├── Mesh_A_2_F2E8FE.uasset
│   │   │   │   │   ├── Mesh_A_2_F2FEC3.uasset
│   │   │   │   │   ├── Mesh_A_2_F46025.uasset
│   │   │   │   │   ├── Mesh_A_2_F4E01C.uasset
│   │   │   │   │   ├── Mesh_A_2_F70257.uasset
│   │   │   │   │   ├── Mesh_A_2_F7643F.uasset
│   │   │   │   │   ├── Mesh_A_2_F7BB75.uasset
│   │   │   │   │   ├── Mesh_A_2_F7DEF3.uasset
│   │   │   │   │   ├── Mesh_A_2_F81362.uasset
│   │   │   │   │   ├── Mesh_A_2_F989A4.uasset
│   │   │   │   │   ├── Mesh_A_2_FA0294.uasset
│   │   │   │   │   ├── Mesh_A_2_FABDB7.uasset
│   │   │   │   │   ├── Mesh_A_2_FB9BFB.uasset
│   │   │   │   │   ├── Mesh_A_2_FD27FC.uasset
│   │   │   │   │   ├── Mesh_A_2_FD9682.uasset
│   │   │   │   │   ├── Mesh_A_2_FDE792.uasset
│   │   │   │   │   └── Mesh_A_2_FFFDB7.uasset
│   │   │   │   ├── Expanse_Blockout/
│   │   │   │   │   ├── Block_1x1m_A_AE19EE.uasset
│   │   │   │   │   ├── Block_1x1m_A_AE19EE2.uasset
│   │   │   │   │   ├── BridgeC_0FB30D.uasset
│   │   │   │   │   ├── BridgeC_2A4BC0.uasset
│   │   │   │   │   ├── BridgeC_F435F8.uasset
│   │   │   │   │   ├── BridgeD_745AEE.uasset
│   │   │   │   │   ├── BridgeE_BE2BF9.uasset
│   │   │   │   │   ├── BridgeF_A24598.uasset
│   │   │   │   │   ├── BridgeG_24FE19.uasset
│   │   │   │   │   ├── BridgeG_8A4B12.uasset
│   │   │   │   │   ├── BridgeH_79E7F1.uasset
│   │   │   │   │   ├── BridgeI_F882AC.uasset
│   │   │   │   │   ├── BridgePlatformA_5DAD14.uasset
│   │   │   │   │   ├── CapsulePanelA_3EE9F1.uasset
│   │   │   │   │   ├── Curve90degA_EDADA0.uasset
│   │   │   │   │   ├── CurvedCornerA_79A789.uasset
│   │   │   │   │   ├── CurvedCornerB_A1B0FC.uasset
│   │   │   │   │   ├── CurvedCornerD_B066FD.uasset
│   │   │   │   │   ├── CurvedCornerE_CDF25B.uasset
│   │   │   │   │   ├── CurvedCornerF_0E6296.uasset
│   │   │   │   │   ├── CurvedCornerH_B658E9.uasset
│   │   │   │   │   ├── CurvedCornerI_9DE3E7.uasset
│   │   │   │   │   ├── CurvedCornerJ_00A6B8.uasset
│   │   │   │   │   ├── CurvedCornerK_E291AD.uasset
│   │   │   │   │   ├── CurvedCornerL_0E20B9.uasset
│   │   │   │   │   ├── CurvedCornerM_37DFB0.uasset
│   │   │   │   │   ├── CurvedCornerN_21DC0E.uasset
│   │   │   │   │   ├── CurvedFrameA_CED960.uasset
│   │   │   │   │   ├── CurvedFrameB_A96453.uasset
│   │   │   │   │   ├── CurvedFrameC_BBCBA1.uasset
│   │   │   │   │   ├── CurvedFrameF_D58171.uasset
│   │   │   │   │   ├── CurvedFrameI_70EB60.uasset
│   │   │   │   │   ├── CurvedFrameK_A6442D.uasset
│   │   │   │   │   ├── CurvedPlatformA_44F777.uasset
│   │   │   │   │   ├── ElevatorShaftA_15FEAE.uasset
│   │   │   │   │   ├── ElevatorShaftA_6D6664.uasset
│   │   │   │   │   ├── ElevatorShaftA_ABC8C9.uasset
│   │   │   │   │   ├── ElevatorShaftB_9B1610.uasset
│   │   │   │   │   ├── ElevatorShaftC_3ED496.uasset
│   │   │   │   │   ├── FloorC_2EE61D.uasset
│   │   │   │   │   ├── FloorE_0861E5.uasset
│   │   │   │   │   ├── MeshA_069AB9.uasset
│   │   │   │   │   ├── MeshA_09FE63.uasset
│   │   │   │   │   ├── MeshA_0AEAE9.uasset
│   │   │   │   │   ├── MeshA_0F7922.uasset
│   │   │   │   │   ├── MeshA_11A71C.uasset
│   │   │   │   │   ├── MeshA_210DB9.uasset
│   │   │   │   │   ├── MeshA_261D01.uasset
│   │   │   │   │   ├── MeshA_2D5981.uasset
│   │   │   │   │   ├── MeshA_45E13F.uasset
│   │   │   │   │   ├── MeshA_51846C.uasset
│   │   │   │   │   ├── MeshA_5668F6.uasset
│   │   │   │   │   ├── MeshA_57F8E0.uasset
│   │   │   │   │   ├── MeshA_651DD3.uasset
│   │   │   │   │   ├── MeshA_6580D3.uasset
│   │   │   │   │   ├── MeshA_6619EC.uasset
│   │   │   │   │   ├── MeshA_68DC05.uasset
│   │   │   │   │   ├── MeshA_6D7B2B.uasset
│   │   │   │   │   ├── MeshA_8FDDCE.uasset
│   │   │   │   │   ├── MeshA_9F59D5.uasset
│   │   │   │   │   ├── MeshA_B007D8.uasset
│   │   │   │   │   ├── MeshA_BB4365.uasset
│   │   │   │   │   ├── MeshA_BE0B22.uasset
│   │   │   │   │   ├── MeshA_F7E137.uasset
│   │   │   │   │   ├── MeshA_F9C7A5.uasset
│   │   │   │   │   ├── MixedCornerPanelA_983A72.uasset
│   │   │   │   │   ├── MixedCornerPanelB_F18C6B.uasset
│   │   │   │   │   ├── PanelFloorA_9F0EC2.uasset
│   │   │   │   │   ├── PanelFloorB_2EEFC7.uasset
│   │   │   │   │   ├── PanelFloorC_38C6C5.uasset
│   │   │   │   │   ├── PanelFloorC_3E49C5.uasset
│   │   │   │   │   ├── PanelFloorD_D3F505.uasset
│   │   │   │   │   ├── PanelPillA_11DB64.uasset
│   │   │   │   │   ├── RampA_084DE2.uasset
│   │   │   │   │   ├── RampA_1402A7.uasset
│   │   │   │   │   ├── RampA_51A1CB.uasset
│   │   │   │   │   ├── RampA_6B19EF.uasset
│   │   │   │   │   ├── RampA_6C5848.uasset
│   │   │   │   │   ├── RoundedBlockA_4375F3.uasset
│   │   │   │   │   ├── SM_BridgeH_1E9B32.uasset
│   │   │   │   │   ├── SM_BridgeH_7DB141.uasset
│   │   │   │   │   ├── SM_CurvedFloorPatchA_C1C156.uasset
│   │   │   │   │   ├── SM_CurvedFloorPatchB_C82BD3.uasset
│   │   │   │   │   ├── SM_FloorD_968DF8.uasset
│   │   │   │   │   ├── SM_TeleporterFrameA_3EC34D.uasset
│   │   │   │   │   ├── SM_TeleporterFrameA_92CAB8.uasset
│   │   │   │   │   ├── SM_TeleporterFrameB_4E67A5.uasset
│   │   │   │   │   ├── SM_WallJ_5BB90A.uasset
│   │   │   │   │   ├── SkyBridgeTrimeA_DD17F1.uasset
│   │   │   │   │   ├── StairsA_C2055F.uasset
│   │   │   │   │   ├── StairsB_12305C.uasset
│   │   │   │   │   ├── StairsD_09167C.uasset
│   │   │   │   │   ├── StairsE_0D5A82.uasset
│   │   │   │   │   ├── WallB_77E966.uasset
│   │   │   │   │   ├── WallC_C7EBFC.uasset
│   │   │   │   │   ├── WallD_B1D74E.uasset
│   │   │   │   │   ├── WallF_81EBEC.uasset
│   │   │   │   │   ├── WallG_12E713.uasset
│   │   │   │   │   ├── WallG_1C05FB.uasset
│   │   │   │   │   ├── WallH_3749AE.uasset
│   │   │   │   │   ├── WallH_457ECD.uasset
│   │   │   │   │   ├── WallL_2DDDBA.uasset
│   │   │   │   │   ├── WallM_6F701A.uasset
│   │   │   │   │   ├── WallN_CC89C4.uasset
│   │   │   │   │   ├── WallO_5E167F.uasset
│   │   │   │   │   ├── WallP_4BBE39.uasset
│   │   │   │   │   ├── WallT_1C5B93.uasset
│   │   │   │   │   ├── WallV_52386B.uasset
│   │   │   │   │   ├── WallW_ADBE4B.uasset
│   │   │   │   │   ├── WindowFrameB_D9E5E1.uasset
│   │   │   │   │   └── WindowFrameC_0EB4BF.uasset
│   │   │   │   ├── FiringRange/
│   │   │   │   │   ├── SM_ModuleA_090.uasset
│   │   │   │   │   ├── SM_Railing_010.uasset
│   │   │   │   │   └── SM_stairs_1.uasset
│   │   │   │   └── UELogo.uasset
│   │   │   ├── System/
│   │   │   │   └── Playlists/
│   │   │   │       ├── DA_Convolution_ControlPoint.uasset
│   │   │   │       ├── DA_Expanse_TDM.uasset
│   │   │   │       ├── T_UI_LoadingScreen_ControlGame.uasset
│   │   │   │       ├── T_UI_LoadingScreen_ShooterGame.uasset
│   │   │   │       ├── W_LoadingScreen_ControlPoint.uasset
│   │   │   │       └── W_LoadingScreen_TDM.uasset
│   │   │   ├── ShooterMaps.uasset
│   │   │   └── ShooterMaps_Label.uasset
│   │   ├── Resources/
│   ├── ShooterTests/
│   │   ├── Content/
│   │   │   ├── Blueprint/
│   │   │   │   ├── BP_GameplayEffectPad_Forcefeedback.uasset
│   │   │   │   ├── B_DevicePropertyTester.uasset
│   │   │   │   ├── B_Test_AutoRun.uasset
│   │   │   │   ├── B_Test_Base.uasset
│   │   │   │   └── B_Test_FireWeapon.uasset
│   │   │   ├── Input/
│   │   │   │   ├── DeviceColor/
│   │   │   │   │   ├── B_InputDeviceColor.uasset
│   │   │   │   │   ├── C_InputDeviceColor.uasset
│   │   │   │   │   ├── FFE_ChangeColorOverTime.uasset
│   │   │   │   │   └── FFE_ChangeColorTest.uasset
│   │   │   │   ├── SoundVibrations/
│   │   │   │   │   ├── B_AudioBasedVibration.uasset
│   │   │   │   │   ├── Emote_FingerGuns_Test.uasset
│   │   │   │   │   ├── GamepadAudioSubmix.uasset
│   │   │   │   │   └── GamepadVibrationSubmix.uasset
│   │   │   │   ├── TriggerFeedback/
│   │   │   │   │   ├── B_TriggerFeedbackTest.uasset
│   │   │   │   │   ├── C_TriggerPosition.uasset
│   │   │   │   │   └── C_TriggerStr.uasset
│   │   │   │   ├── TriggerVibration/
│   │   │   │   │   ├── B_Trigger_Vibration_Test.uasset
│   │   │   │   │   ├── C_TriggerVibration_Amplitude.uasset
│   │   │   │   │   ├── C_TriggerVibration_Frequency.uasset
│   │   │   │   │   └── C_TriggerVibration_Position.uasset
│   │   │   │   ├── C_InputDeviceColor.uasset
│   │   │   │   └── FFE_ChangeColorTest.uasset
│   │   │   ├── Maps/
│   │   │   │   ├── L_ShooterTest_AutoRun.umap
│   │   │   │   ├── L_ShooterTest_Basic.umap
│   │   │   │   ├── L_ShooterTest_DeviceProperties.umap
│   │   │   │   └── L_ShooterTest_FireWeapon.umap
│   │   │   ├── System/
│   │   │   │   └── Experiences/
│   │   │   │       ├── B_AutomatedShooterTest.uasset
│   │   │   │       └── B_BasicShooterTest.uasset
│   │   │   └── ShooterTests.uasset
│   │   ├── Resources/
│   │   ├── Source/
│   │   │   └── ShooterTestsRuntime/
│   │   │       ├── Private/
│   │   │       │   ├── Utilities/
│   └── TopDownArena/
│       ├── Config/
│       │   └── Tags/
│       ├── Content/
│       │   ├── Game/
│       │   │   ├── Bombs/
│       │   │   │   ├── B_Bomb_Base.uasset
│       │   │   │   ├── B_Bomb_Standard.uasset
│       │   │   │   ├── BombPulseTime.uasset
│       │   │   │   ├── Cue_DropBomb.uasset
│       │   │   │   ├── DropBomb_Montage.uasset
│       │   │   │   ├── Explosion22.uasset
│       │   │   │   ├── ExplosionFireballCue.uasset
│       │   │   │   ├── GA_DropBomb.uasset
│       │   │   │   ├── GE_Damaged_By_Bomb.uasset
│       │   │   │   ├── GE_DecrementBombsRemaining.uasset
│       │   │   │   ├── GE_IncrementBombsRemaining.uasset
│       │   │   │   ├── M_Bomb.uasset
│       │   │   │   ├── M_Fireball.uasset
│       │   │   │   └── TestCurveAtlas.uasset
│       │   │   ├── Effects/
│       │   │   │   ├── Bomb/
│       │   │   │   │   ├── M_ExplosionMaterial.uasset
│       │   │   │   │   └── NS_BombExplosion.uasset
│       │   │   │   ├── PickUp/
│       │   │   │   │   ├── M_Bomb.uasset
│       │   │   │   │   ├── M_VerticalGradient.uasset
│       │   │   │   │   ├── M_WoodBoxPickUp.uasset
│       │   │   │   │   ├── NS_TopDownArenaPickupBomb.uasset
│       │   │   │   │   ├── NS_TopDownArenaPickupRange.uasset
│       │   │   │   │   ├── SM_CylinderVertical_Effect.uasset
│       │   │   │   │   ├── SM_ExtraBomb.uasset
│       │   │   │   │   ├── SM_ExtraRange.uasset
│       │   │   │   │   └── T_BombPickUp.uasset
│       │   │   │   ├── Player/
│       │   │   │   │   └── M_PlayerCircle.uasset
│       │   │   │   ├── M_Sparks_No_AA.uasset
│       │   │   │   ├── NS_BombFuse.uasset
│       │   │   │   ├── N_Emitter_Flame.uasset
│       │   │   │   ├── N_Emitter_MuzzleFlash_Smoke.uasset
│       │   │   │   └── SM_GeoSphere.uasset
│       │   │   ├── Environment/
│       │   │   │   ├── B_DestructableBlock.uasset
│       │   │   │   ├── B_EditorMapInfo.uasset
│       │   │   │   ├── MI_DestructibleBlock.uasset
│       │   │   │   ├── MI_GroundFloor.uasset
│       │   │   │   ├── M_Background.uasset
│       │   │   │   └── M_GroundFloor.uasset
│       │   │   ├── MapsGenerator/
│       │   │   │   ├── B_BlockSpawning.uasset
│       │   │   │   └── B_LevelGeneration.uasset
│       │   │   ├── Modes/
│       │   │   │   ├── B_TopDownArena_GameComponent_Base.uasset
│       │   │   │   ├── B_TopDownArena_GameComponent_Lives.uasset
│       │   │   │   ├── B_TopDownArena_GameComponent_SpawnLocalPlayers.uasset
│       │   │   │   ├── GE_Warmup.uasset
│       │   │   │   ├── Phase_Playing.uasset
│       │   │   │   ├── Phase_PostGame.uasset
│       │   │   │   └── Phase_Warmup.uasset
│       │   │   ├── Pickups/
│       │   │   │   ├── B_TopDownArena_Pickup.uasset
│       │   │   │   ├── GET_ArenaPickup_Base.uasset
│       │   │   │   ├── GE_AdditionalHeart.uasset
│       │   │   │   ├── GE_BombCountUp.uasset
│       │   │   │   ├── GE_BombRangeUp.uasset
│       │   │   │   ├── GE_MoveSpeedUp.uasset
│       │   │   │   ├── T_BombCountUp.uasset
│       │   │   │   ├── T_BombRangeUp.uasset
│       │   │   │   ├── T_HeartUp.uasset
│       │   │   │   └── T_SpeedUp.uasset
│       │   │   ├── Powerups/
│       │   │   │   ├── GE_Stat_BombCount.uasset
│       │   │   │   ├── GE_Stat_FireRange.uasset
│       │   │   │   └── GE_Stat_MoveSpeed.uasset
│       │   │   ├── AbilitySet_Arena.uasset
│       │   │   ├── B_Hero_Arena.uasset
│       │   │   ├── B_TeamSetup_FourTeams.uasset
│       │   │   ├── CM_ArenaFramingCamera.uasset
│       │   │   ├── GA_ArenaHero_Death.uasset
│       │   │   ├── HeroData_Arena.uasset
│       │   │   └── InputData_Arena.uasset
│       │   ├── GameplayCues/
│       │   │   ├── GCN_GetReady.uasset
│       │   │   ├── GCN_PickupAcquired.uasset
│       │   │   └── W_GetReady.uasset
│       │   ├── Maps/
│       │   │   ├── L_TopDownArenaGym.umap
│       │   │   └── L_TopDown_LocalMultiplayer.umap
│       │   ├── System/
│       │   │   ├── Experiences/
│       │   │   │   ├── B_TopDownArenaExperience.uasset
│       │   │   │   └── B_TopDownArena_Multiplayer_Experience.uasset
│       │   │   └── Playlists/
│       │   │       ├── DA_TopDownArena_Multiplayer.uasset
│       │   │       └── DA_TopDownArena_Splode.uasset
│       │   ├── UserInterface/
│       │   │   ├── Player/
│       │   │   │   ├── Texture/
│       │   │   │   │   ├── T_Bombs.uasset
│       │   │   │   │   ├── T_Health_Icon.uasset
│       │   │   │   │   ├── T_Range.uasset
│       │   │   │   │   └── T_Speed.uasset
│       │   │   │   ├── B_PlayerUIComponent.uasset
│       │   │   │   ├── W_PlayerTile.uasset
│       │   │   │   └── W_Powerup_Stat.uasset
│       │   │   ├── W_RoundTimer.uasset
│       │   │   ├── W_TopDownArenaHUDLayout.uasset
│       │   │   └── W_TopDownPlayers.uasset
│       │   ├── TopDownArena.uasset
│       │   └── TopDownArena_Label.uasset
│       ├── Resources/
│       ├── Source/
│       │   └── TopDownArenaRuntime/
│       │       ├── Private/
│       │       ├── Public/
├── GameSettings/
│   ├── Source/
│   │   ├── Private/
│   │   │   ├── DataSource/
│   │   │   ├── EditCondition/
│   │   │   ├── Registry/
│   │   │   ├── Widgets/
│   │   │   │   ├── Misc/
│   │   │   │   ├── Responsive/
│   │   ├── Public/
│   │   │   ├── DataSource/
│   │   │   ├── EditCondition/
│   │   │   ├── Widgets/
│   │   │   │   ├── Misc/
├── GameSubtitles/
│   ├── Config/
│   │   └── Tags/
│   ├── Source/
│   │   ├── Private/
│   │   │   ├── Players/
│   │   │   ├── Widgets/
│   │   ├── Public/
│   │   │   ├── Players/
│   │   │   ├── Widgets/
├── GameplayMessageRouter/
│   ├── Source/
│   │   ├── GameplayMessageNodes/
│   │   │   ├── Private/
│   │   │   ├── Public/
│   │   └── GameplayMessageRuntime/
│   │       ├── Private/
│   │       │   └── GameFramework/
│   │       ├── Public/
│   │       │   └── GameFramework/
├── LyraExampleContent/
│   ├── Content/
│   │   ├── MSPresets/
│   │   │   ├── MSTextures/
│   │   │   │   ├── BlackPlaceholder.uasset
│   │   │   │   ├── Placeholder.uasset
│   │   │   │   ├── Placeholder_Normal.uasset
│   │   │   │   └── WhitePlaceholder.uasset
│   │   │   └── M_MS_Surface_Material/
│   │   │       ├── Functions/
│   │   │       │   ├── MF_Displacement.uasset
│   │   │       │   ├── MF_MapAdjustments.uasset
│   │   │       │   ├── MF_ObjAdjustments.uasset
│   │   │       │   ├── MF_Tiling.uasset
│   │   │       │   └── MF_Transmission.uasset
│   │   │       ├── SubSurfaceProfile/
│   │   │       │   ├── Red.uasset
│   │   │       │   └── blue.uasset
│   │   │       ├── MI_SurfaceGridBase.uasset
│   │   │       └── M_MS_Surface_Material_TriPlanar.uasset
│   │   ├── Materials/
│   │   │   ├── MF/
│   │   │   │   ├── MF_2DGrid.uasset
│   │   │   │   ├── MF_NormalStrength.uasset
│   │   │   │   ├── MF_TripleGrid.uasset
│   │   │   │   ├── MF_WorldGrid.uasset
│   │   │   │   └── WorldAlignedTextureMip.uasset
│   │   │   ├── Textures/
│   │   │   │   ├── GlassPattern/
│   │   │   │   │   ├── T_GlassTiles_N_1.uasset
│   │   │   │   │   ├── T_Glass_Normal_Tiling.uasset
│   │   │   │   │   ├── T_Glass_Opacity_Tiled.uasset
│   │   │   │   │   └── T_Glass_Pattern_Option_05.uasset
│   │   │   │   ├── T_Concrete_Diffuse.uasset
│   │   │   │   ├── T_Concrete_Glossiness.uasset
│   │   │   │   ├── T_Concrete_Normal.uasset
│   │   │   │   ├── T_GrungySurface_slnneipc_4K_MR.uasset
│   │   │   │   ├── T_Paint_Diffuse.uasset
│   │   │   │   ├── T_Paint_Glossiness.uasset
│   │   │   │   ├── T_Paint_Normal.uasset
│   │   │   │   ├── T_Paint_Opacity.uasset
│   │   │   │   ├── T_ReflectorHexagon_N.uasset
│   │   │   │   ├── T_Rough_Concrete_Floor_ulmmbiuo_4K_N.uasset
│   │   │   │   └── T_StainlessBrushed_2K_normal.uasset
│   │   │   ├── MI_Black_GlossFakeFloor.uasset
│   │   │   ├── MI_GlassA_1.uasset
│   │   │   ├── MI_GlassA_2.uasset
│   │   │   ├── MI_GlassA_3.uasset
│   │   │   ├── MI_GlassB_1.uasset
│   │   │   ├── MI_GlassEdge.uasset
│   │   │   ├── MI_Glass_Panels_2.uasset
│   │   │   ├── MI_Glass_Tile.uasset
│   │   │   ├── MI_Glass_Tile1.uasset
│   │   │   ├── MI_MS_Basic_Metal.uasset
│   │   │   ├── MI_MS_Blue_1.uasset
│   │   │   ├── MI_MS_Blue_2.uasset
│   │   │   ├── MI_MS_Orange_1.uasset
│   │   │   ├── MI_MS_Red_1.uasset
│   │   │   ├── MI_MS_Surface_1.uasset
│   │   │   ├── MI_MS_Surface_12.uasset
│   │   │   ├── MI_MS_Surface_13.uasset
│   │   │   ├── MI_MS_Surface_14.uasset
│   │   │   ├── MI_MS_Surface_2.uasset
│   │   │   ├── MI_MS_Surface_3.uasset
│   │   │   ├── MI_MS_Surface_4.uasset
│   │   │   ├── MI_MS_Surface_5.uasset
│   │   │   ├── MI_MS_Surface_6.uasset
│   │   │   ├── MI_MS_Surface_7.uasset
│   │   │   ├── MI_MS_Surface_8.uasset
│   │   │   ├── MI_MS_Surface_Wall_1.uasset
│   │   │   ├── MI_MS_Surface_Wall_2.uasset
│   │   │   ├── MI_MS_Surface_Wall_3.uasset
│   │   │   ├── MI_MetalChroma.uasset
│   │   │   ├── MI_Tile_1.uasset
│   │   │   ├── MI_tintedGlassTile_Y.uasset
│   │   │   ├── MPC_Checker.uasset
│   │   │   ├── M_GlassA.uasset
│   │   │   ├── M_GlassB.uasset
│   │   │   ├── M_GlassEdge.uasset
│   │   │   ├── M_GlassTinted.uasset
│   │   │   ├── M_Glass_Panels_NM.uasset
│   │   │   ├── M_Glass_Tile.uasset
│   │   │   ├── M_Metal.uasset
│   │   │   ├── M_MetalChroma.uasset
│   │   │   └── M_Tile.uasset
│   │   └── Megascans/
│   │       └── Surfaces/
│   │           ├── Dirty_Painted_Concrete_Wall_vdqfcg3/
│   │           │   ├── T_DirtyPaintedConcreteWall_vdqfcg3_4K_ORDp.uasset
│   │           │   ├── T_Dirty_Painted_Concrete_Wall_vdqfcg3_4K_D.uasset
│   │           │   └── T_Dirty_Painted_Concrete_Wall_vdqfcg3_4K_N.uasset
│   │           └── Fortified_Shelter_Wall_ubimbhrdy/
│   │               └── T_FortifiedShelterWall_ubimbhrdy_4K_ORDp.uasset
├── LyraExtTool/
│   ├── Content/
│   │   └── EUW_MaterialTool.uasset
│   ├── Resources/
│   ├── Source/
│   │   └── LyraExtTool/
│   │       ├── Private/
│   │       ├── Public/
├── ModularGameplayActors/
│   ├── Source/
│   │   └── ModularGameplayActors/
│   │       ├── Private/
│   │       ├── Public/
├── PocketWorlds/
│   ├── Content/
│   │   ├── M_Black.uasset
│   │   ├── M_PocketCaptureMasked.uasset
│   │   └── M_White.uasset
│   ├── Source/
│   │   ├── Private/
│   │   ├── Public/
└── UIExtension/
    ├── Source/
    │   ├── Private/
    │   │   ├── Widgets/
    │   ├── Public/
    │   │   ├── Widgets/
```