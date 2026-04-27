# UE5_Lyra学习指南_006_Build文件

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
- [UE5\_Lyra学习指南\_006\_Build文件](#ue5_lyra学习指南_006_build文件)
	- [创建项目所有的源文件和头文件](#创建项目所有的源文件和头文件)
	- [完善Build文件](#完善build文件)
		- [LyraEditor](#lyraeditor)
			- [1. 核心模块依赖（PublicDependencyModuleNames）](#1-核心模块依赖publicdependencymodulenames)
			- [2. 私有依赖模块（PrivateDependencyModuleNames）](#2-私有依赖模块privatedependencymodulenames)
			- [3. 动态加载模块（DynamicallyLoadedModuleNames）](#3-动态加载模块dynamicallyloadedmodulenames)
			- [4. 关键编译配置](#4-关键编译配置)
			- [模块的主要作用总结](#模块的主要作用总结)
		- [LyraGame](#lyragame)
			- [1. 模块类型与核心依赖](#1-模块类型与核心依赖)
			- [2. 公有依赖模块（PublicDependencyModuleNames）](#2-公有依赖模块publicdependencymodulenames)
			- [3. 私有依赖模块（PrivateDependencyModuleNames）](#3-私有依赖模块privatedependencymodulenames)
			- [4. 动态加载模块（DynamicallyLoadedModuleNames）](#4-动态加载模块dynamicallyloadedmodulenames)
			- [5. 关键编译配置与功能开关](#5-关键编译配置与功能开关)
			- [模块的核心作用总结](#模块的核心作用总结)
		- [补充部分模块](#补充部分模块)
			- [1. PropertyPath](#1-propertypath)
			- [2. RHI (Rendering Hardware Interface)](#2-rhi-rendering-hardware-interface)
			- [3. Gauntlet](#3-gauntlet)
			- [4. ClientPilot](#4-clientpilot)
			- [总结：这些模块如何服务LyraGame\*\*](#总结这些模块如何服务lyragame)
			- [关联性问题思考](#关联性问题思考)
	- [补充一下路径检索](#补充一下路径检索)
	- [总结](#总结)

本节核心步骤,创建源文件和头文件,拷贝插件,修改.uproject文件,修改.Build.cs文件.....
但是有些细节仍然值得我们去探讨一下.

## 创建项目所有的源文件和头文件

为了方便我们专注地编写代码,此时直接将项目所有的UEC++文件创建完成.包括其头文件和源文件.
一般插件的头文件和源文件,我们将直接导入.后续将在其他章节讲解.
GameFeature的插件头文件和源文件,我们将随着课程进度推进而择机编写.

此处注意细节:
1.每个文件头一行需要编写版权声明
2.头文件第二行为空格,第三行为#pragma once
3.源文件第二行为空格,第三行为其对应的头文件

头文件:
``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once
```
源文件
``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.

#include "xxxx.h"
```
可以将项目自行编写的代码首行注释的版权申明更换为自己的如下
``` cpp
// Copyright 2025 Huagang Liu. All Rights Reserved.
```
虚幻引擎创建新的UEC++文件的方式是多种多样的:
1.通过编辑器创建
2.通过IDEA创建
3.通过创建.txt改名刷新项目创建.

我已经将创建好的空白文件整理好了.
大家只需要将工程目录下的文件夹拷贝过去即可.没必要手动去创建每个文件.节约时间.
本章结束时,亦会提供一个课程示例工大家使用.大家也可以通过git版本管理查看.
如果有啥疏漏的,后续再补充即可.

## 完善Build文件
本节需要完善两个Build文件,分别是LyraEditor.Build.cs文件和LyraGameBuild.cs文件.
此处注意细节:
1.需要将所有插件由示例工程Lyra插件目录拷贝到我们自己的工程Lyra同级目录下插件目录.
因为我们的模块会使用到插件.但是不要拷贝GameFeatures文件夹.这些游戏特性插件,我们需要自行编写.

2.需要在Lyra.uproject中开启对应的插件.并关闭GameFeatures插件.因为我们目前还没有.
一共需要关闭是5个如下:
ShooterCore
ShooterMaps
TopDownArena
ShooterExplorer
ShooterTests

此时我们的工程可以正常编译并启动!
但是日志有报错.
我们不在这里处理这些报错.这是因为有些扫描的资产类型及路径,或者ini文件没有配置导致的.
``` txt
Error        LogGameFeatures           Asset manager settings do not include a rule for assets of type GameFeatureData, which is required for game feature plugins to function
Error        LoadErrors                Asset Manager settings do not include an entry for assets of type GameFeatureData, which is required for game feature plugins to function. Add entry to PrimaryAssetTypesToScan?
Error        LoadErrors                Collision Profile settings do not include an entry for the Water Body Collision profile, which is required for water collision to function. Add entry to DefaultEngine.ini?

```
此处给出我们当前使用的uproject文件:

``` json
{
	"FileVersion": 3,
	"EngineAssociation": "5.6",
	"Category": "XG",
	"Description": "由小刚进行复刻的Lyra工程项目",
	"Modules": [
		{
			"Name": "LyraGame",
			"Type": "Runtime",
			"LoadingPhase": "Default",
			"AdditionalDependencies": [
				"DeveloperSettings",
				"Engine"
			]
		},
		{
			"Name": "LyraEditor",
			"Type": "Editor",
			"LoadingPhase": "Default"
		}
	],
	"Plugins": [
		{
			"Name": "ActorPalette",
			"Enabled": true
		},
		{
			"Name": "AESGCMHandlerComponent",
			"Enabled": true
		},
		{
			"Name": "DTLSHandlerComponent",
			"Enabled": true
		},
		{
			"Name": "GameplayAbilities",
			"Enabled": true
		},
		{
			"Name": "Gauntlet",
			"Enabled": true
		},
		{
			"Name": "D3DExternalGPUStatistics",
			"Enabled": true,
			"Optional": true,
			"SupportedTargetPlatforms": [
				"Win64"
			]
		},
		{
			"Name": "CommonLoadingScreen",
			"Enabled": true
		},
		{
			"Name": "CommonConversation",
			"Enabled": true
		},
		{
			"Name": "GameFeatures",
			"Enabled": true
		},
		{
			"Name": "ModularGameplay",
			"Enabled": true
		},
		{
			"Name": "ModularGameplayActors",
			"Enabled": true
		},
		{
			"Name": "EnhancedInput",
			"Enabled": true
		},
		{
			"Name": "WinDualShock",
			"Enabled": true,
			"SupportedTargetPlatforms": [
				"Win64"
			]
		},
		{
			"Name": "Volumetrics",
			"Enabled": true
		},
		{
			"Name": "DataRegistry",
			"Enabled": true
		},
		{
			"Name": "ReplicationGraph",
			"Enabled": true
		},
		{
			"Name": "SignificanceManager",
			"Enabled": true
		},
		{
			"Name": "Niagara",
			"Enabled": true
		},
		{
			"Name": "Water",
			"Enabled": true
		},
		{
			"Name": "CommonUI",
			"Enabled": true
		},
		{
			"Name": "ControlFlows",
			"Enabled": true
		},
		{
			"Name": "GameSettings",
			"Enabled": true
		},
		{
			"Name": "CommonUser",
			"Enabled": true
		},
		{
			"Name": "CommonGame",
			"Enabled": true
		},
		{
			"Name": "GameSubtitles",
			"Enabled": true
		},
		{
			"Name": "PocketWorlds",
			"Enabled": true
		},
		{
			"Name": "UIExtension",
			"Enabled": true
		},
		{
			"Name": "AsyncMixin",
			"Enabled": true
		},
		{
			"Name": "Metasound",
			"Enabled": true
		},
		{
			"Name": "MagicLeap",
			"Enabled": false
		},
		{
			"Name": "MagicLeapMedia",
			"Enabled": false
		},
		{
			"Name": "MagicLeapPassableWorld",
			"Enabled": false
		},
		{
			"Name": "OpenXREyeTracker",
			"Enabled": false
		},
		{
			"Name": "OpenXRHandTracking",
			"Enabled": false
		},
		{
			"Name": "OpenXRHMD",
			"Enabled": false
		},
		{
			"Name": "SteamVR",
			"Enabled": false
		},
		{
			"Name": "GearVR",
			"Enabled": false
		},
		{
			"Name": "LuminPlatformFeatures",
			"Enabled": false,
			"SupportedTargetPlatforms": [
				"Lumin"
			]
		},
		{
			"Name": "MLSDK",
			"Enabled": false
		},
		{
			"Name": "OnlineFramework",
			"Enabled": true
		},
		{
			"Name": "PlayFabParty",
			"Enabled": true,
			"PlatformAllowList": [
				"XB1",
				"XSX",
				"WinGDK"
			],
			"SupportedTargetPlatforms": [
				"XB1",
				"XSX",
				"WinGDK"
			]
		},
		{
			"Name": "EOSReservedHooks",
			"Enabled": true,
			"Optional": true
		},
		{
			"Name": "OnlineSubsystemEOS",
			"Enabled": true
		},
		{
			"Name": "OnlineServicesEOS",
			"Enabled": true
		},
		{
			"Name": "OnlineServicesNull",
			"Enabled": true
		},
		{
			"Name": "OnlineServicesOSSAdapter",
			"Enabled": true
		},
		{
			"Name": "OnlineSubsystemSteam",
			"Enabled": true
		},
		{
			"Name": "SocketSubsystemSteamIP",
			"Enabled": true
		},
		{
			"Name": "GameplayMessageRouter",
			"Enabled": true
		},
		{
			"Name": "SteamSockets",
			"Enabled": true
		},
		{
			"Name": "AssetReferenceRestrictions",
			"Enabled": true
		},
		{
			"Name": "ModelingToolsEditorMode",
			"Enabled": true
		},
		{
			"Name": "GeometryScripting",
			"Enabled": true
		},
		{
			"Name": "AnimationLocomotionLibrary",
			"Enabled": true
		},
		{
			"Name": "AudioModulation",
			"Enabled": true
		},
		{
			"Name": "AudioGameplayVolume",
			"Enabled": true
		},
		{
			"Name": "AudioGameplay",
			"Enabled": true
		},
		{
			"Name": "SoundUtilities",
			"Enabled": true
		},
		{
			"Name": "AnimationWarping",
			"Enabled": true
		},
		{
			"Name": "MovieRenderPipeline",
			"Enabled": true,
			"TargetAllowList": [
				"Editor"
			]
		},
		{
			"Name": "MoviePipelineMaskRenderPass",
			"Enabled": true,
			"TargetAllowList": [
				"Editor"
			]
		},
		{
			"Name": "AssetSearch",
			"Enabled": true
		},
		{
			"Name": "GameplayInsights",
			"Enabled": true
		},
		{
			"Name": "ResonanceAudio",
			"Enabled": false
		},
		{
			"Name": "RuntimePhysXCooking",
			"Enabled": false
		},
		{
			"Name": "Spatialization",
			"Enabled": true
		},
		{
			"Name": "ShooterCore",
			"Enabled": false
		},
		{
			"Name": "ShooterMaps",
			"Enabled": false
		},
		{
			"Name": "TopDownArena",
			"Enabled": false
		},
		{
			"Name": "FunctionalTestingEditor",
			"Enabled": true
		},
		{
			"Name": "ShooterExplorer",
			"Enabled": false
		},
		{
			"Name": "ShooterTests",
			"Enabled": false
		},
		{
			"Name": "GameplayInteractions",
			"Enabled": true
		},
		{
			"Name": "SmartObjects",
			"Enabled": true
		},
		{
			"Name": "ContextualAnimation",
			"Enabled": true
		},
		{
			"Name": "GameplayBehaviorSmartObjects",
			"Enabled": true
		},
		{
			"Name": "GameplayStateTree",
			"Enabled": true
		},
		{
			"Name": "GameplayBehaviors",
			"Enabled": true
		},
		{
			"Name": "RuntimeTests",
			"Enabled": true
		},
		{
			"Name": "AutomatedPerfTesting",
			"Enabled": true
		},
		{
			"Name": "Reflex",
			"Enabled": true
		}
	]
}
```

现在我们完善Build文件.

### LyraEditor

此处直接给出LyraEditor.Build.cs文件的DeepSeek分析.[偷个懒]
相应的注释已补充到工程中.

这段代码是虚幻引擎中 **`LyraEditor` 模块**的构建规则文件（`LyraEditor.Build.cs`），属于 **Lyra 游戏项目**的 **编辑器专用模块**。它定义了模块的依赖项和编译配置，主要用于扩展虚幻编辑器的功能（如自定义工具、游戏数据编辑等）。以下是模块名称及其作用的分类解析：

---

#### 1. 核心模块依赖（PublicDependencyModuleNames）
这些是模块公开依赖的基础功能，其他模块可以间接使用它们。

| 模块名称                     | 作用                                                                 |
|------------------------------|----------------------------------------------------------------------|
| **`Core`**                   | UE 基础类型（字符串、容器等）。                                       |
| **`CoreUObject`**            | UObject 系统（反射、序列化）。                                        |
| **`Engine`**                 | 引擎核心功能（材质、粒子等）。                                        |
| **`EditorFramework`**        | 编辑器扩展框架（自定义资产编辑器等）。                                |
| **`UnrealEd`**               | 虚幻编辑器核心功能（关卡编辑、细节面板等）。                          |
| **`PhysicsCore`**            | 物理系统基础支持。                                                    |
| **`GameplayTagsEditor`**     | 游戏标签（GameplayTags）的编辑器支持（如标签管理界面）。              |
| **`GameplayTasksEditor`**    | 游戏任务系统的编辑器工具。                                            |
| **`GameplayAbilities`**      | 游戏技能系统（GA）的运行时逻辑。                                      |
| **`GameplayAbilitiesEditor`**| GA 系统的编辑器工具（如技能蓝图编辑）。                               |
| **`StudioTelemetry`**        | 开发工作室的遥测数据收集（用于分析开发行为）。                        |
| **`LyraGame`**               | Lyra 游戏本身的运行时模块（依赖游戏逻辑）。                           |

---

#### 2. 私有依赖模块（PrivateDependencyModuleNames）
这些是仅限模块内部使用的功能，不会暴露给其他模块。

| 模块名称                     | 作用                                                                 |
|------------------------------|----------------------------------------------------------------------|
| **`InputCore`**              | 输入设备（键盘、鼠标）支持。                                          |
| **`Slate`/`SlateCore`**      | UE 的 UI 框架（编辑器界面元素）。                                     |
| **`ToolMenus`**              | 编辑器工具栏和菜单扩展。                                              |
| **`EditorStyle`**            | 编辑器UI样式（图标、字体等）。                                        |
| **`DataValidation`**         | 数据校验工具（防止无效资产）。                                        |
| **`MessageLog`**             | 编辑器日志系统。                                                      |
| **`Projects`**               | 项目管理（插件、游戏路径等）。                                        |
| **`DeveloperToolSettings`**  | 开发者工具配置。                                                      |
| **`CollectionManager`**      | 资产集合管理（分类、标签等）。                                        |
| **`SourceControl`**          | 版本控制集成（Git、Perforce等）。                                     |
| **`Chaos`**                  | Chaos 物理引擎支持。                                                  |
| **`ExternalRpcRegistry`**    | 外部 RPC（远程过程调用）框架（非发布版启用，见下文）。                |
| **`HTTPServer`**             | HTTP 服务器功能（仅开发/测试版启用）。                                |

---

#### 3. 动态加载模块（DynamicallyLoadedModuleNames）
当前未动态加载任何模块（列表为空）。

---

#### 4. 关键编译配置
- **RPC 与 HTTP 服务器**  
  - 在 **非发布版本**（`Shipping`）中禁用 RPC 和 HTTP 服务（安全考虑）：  
    ```cpp
    PublicDefinitions.Add("WITH_RPC_REGISTRY=0");
    PublicDefinitions.Add("WITH_HTTPSERVER_LISTENERS=0");
    ```
  - 其他版本启用：  
    ```cpp
    PrivateDependencyModuleNames.Add("HTTPServer");
    PublicDefinitions.Add("WITH_RPC_REGISTRY=1");
    ```
- **调试工具限制**  
  - 禁止在发布版使用 `DrawDebug` 函数：  
    ```cpp
    PublicDefinitions.Add("SHIPPING_DRAW_DEBUG_ERROR=1");
    ```

---

#### 模块的主要作用总结
- **扩展编辑器功能**：为 Lyra 游戏提供专属的编辑器工具（如技能编辑、标签管理）。
- **集成游戏系统**：依赖 `LyraGame` 和 `GameplayAbilities`，实现游戏逻辑与编辑器的联动。
- **安全与调试控制**：通过条件编译限制敏感功能（RPC、Debug）在发布版本中的使用。

如果需要更具体的功能实现分析（如某个依赖模块的实际用途），可以进一步探讨！

### LyraGame

此处直接给出LyraGame.Build.cs文件的DeepSeek分析.[继续偷个懒]
相应的注释已补充到工程中.

以下是 **`LyraGame` 模块**的详细分析，基于你提供的 `LyraGame.Build.cs` 代码。该模块是 **Lyra 游戏项目的核心运行时模块**，负责游戏逻辑、技能系统、网络同步等关键功能，同时集成了多种UE5的高级特性。

---

#### 1. 模块类型与核心依赖
- **类型**：运行时模块（`Runtime`），用于游戏运行时的核心逻辑。
- **PCH 使用**：显式或共享 PCH（优化编译速度）。
- **关键公有路径**：`LyraGame` 目录下的头文件公开给其他模块。

---

#### 2. 公有依赖模块（PublicDependencyModuleNames）
这些是模块公开依赖的基础功能和系统，其他模块可以间接使用它们。

| 模块名称                     | 作用                                                                 |
|------------------------------|----------------------------------------------------------------------|
| **`Core`/`CoreUObject`**     | UE 基础类型和反射系统。                                              |
| **`CoreOnline`**             | 在线功能基础支持（如登录、会话管理）。                               |
| **`ApplicationCore`**        | 应用程序级功能（如窗口管理、输入事件）。                            |
| **`Engine`**                 | 引擎核心（Actor、关卡、渲染等）。                                    |
| **`PhysicsCore`**            | 物理系统基础支持。                                                   |
| **`GameplayTags`**           | 游戏标签系统（用于技能、状态分类）。                                 |
| **`GameplayTasks`**          | 异步任务管理（如AI行为、技能流程）。                                 |
| **`GameplayAbilities`**      | 游戏技能系统（GA）的核心逻辑。                                       |
| **`AIModule`**               | AI 行为树、导航系统。                                                |
| **`ModularGameplay`**        | 模块化游戏设计（动态组合游戏功能）。                                 |
| **`ModularGameplayActors`**  | 支持模块化设计的Actor类。                                            |
| **`DataRegistry`**           | 数据驱动注册表（动态加载配置数据）。                                 |
| **`ReplicationGraph`**       | 优化的网络复制系统（大型地图同步）。                                 |
| **`GameFeatures`**           | 游戏功能插件系统（动态加载DLC/模块）。                               |
| **`Niagara`**                | 粒子特效系统。                                                       |
| **`CommonLoadingScreen`**    | 通用加载屏幕管理。                                                   |
| **`AsyncMixin`**             | 异步操作支持（混合到其他类中）。                                     |
| **`ControlFlows`**           | 复杂逻辑流程控制（如技能链、剧情分支）。                             |

---

#### 3. 私有依赖模块（PrivateDependencyModuleNames）
这些是模块内部使用的功能，不暴露给其他模块。

| 模块名称                     | 作用                                                                 |
|------------------------------|----------------------------------------------------------------------|
| **`InputCore`**              | 输入设备支持。                                                       |
| **`Slate`/`SlateCore`**      | UI 框架基础（非游戏内UI，如编辑器工具）。                            |
| **`EnhancedInput`**          | 增强输入系统（支持复杂输入映射）。                                   |
| **`UMG`**                    | 游戏内UI（用户界面控件）。                                           |
| **`CommonUI`**               | 通用UI组件（如菜单、HUD）。                                          |
| **`CommonInput`**            | 跨平台输入统一管理。                                                 |
| **`CommonGame`**             | 通用游戏逻辑（如玩家管理、游戏模式）。                               |
| **`CommonUser`**             | 用户账户与权限系统。                                                 |
| **`GameplayMessageRuntime`** | 游戏内消息总线（事件广播）。                                         |
| **`AudioMixer`**             | 高级音频混合与控制。                                                 |
| **`NetworkReplayStreaming`** | 网络回放功能（录制与播放游戏回放）。                                 |
| **`UIExtension`**            | 动态UI扩展（如Mod添加菜单项）。                                      |
| **`AudioModulation`**        | 音频动态调节（根据游戏状态改变音效）。                               |
| **`DTLSHandlerComponent`**   | 加密网络通信支持（安全传输层）。                                     |
| **`Json`**                   | JSON 数据解析与序列化。                                              |

---

#### 4. 动态加载模块（DynamicallyLoadedModuleNames）
当前未动态加载任何模块（列表为空），表明所有功能均静态链接。

---

#### 5. 关键编译配置与功能开关
- **RPC 与 HTTP 服务器**  
  - 在 **发布版本（Shipping）** 中禁用：  
    ```cpp
    PublicDefinitions.Add("WITH_RPC_REGISTRY=0");
    PublicDefinitions.Add("WITH_HTTPSERVER_LISTENERS=0");
    ```
  - 其他版本启用：  
    ```cpp
    PrivateDependencyModuleNames.Add("HTTPServer");
    PublicDefinitions.Add("WITH_RPC_REGISTRY=1");
    ```
  - 依赖 `ExternalRpcRegistry` 实现外部远程调用（开发调试用）。  

- **调试工具限制**  
  - 禁止在发布版使用 `DrawDebug`：  
    ```cpp
    PublicDefinitions.Add("SHIPPING_DRAW_DEBUG_ERROR=1");
    ```

- **高级支持启用**  
  - **`SetupGameplayDebuggerSupport`**：集成游戏调试器（如显示AI状态、网络信息）。  
  - **`SetupIrisSupport`**：启用 Iris 网络同步系统（UE5 的新复制框架）。  

---

#### 模块的核心作用总结
1. **游戏逻辑基石**  
   - 通过 `GameplayAbilities` 和 `ModularGameplay` 实现技能系统与模块化设计。  
   - 依赖 `CommonGame`/`CommonUser` 提供玩家管理和账户功能。  

2. **网络与多人支持**  
   - 使用 `ReplicationGraph` 和 `Iris` 优化网络同步。  
   - 集成 `DTLSHandlerComponent` 保障通信安全。  

3. **扩展性与数据驱动**  
   - `GameFeatures` 支持动态功能加载（如DLC）。  
   - `DataRegistry` 实现配置数据动态读取。  

4. **开发与调试工具**  
   - 条件编译限制敏感功能，确保发布版本安全性。  

如果需要进一步分析某个子系统的实现细节（如 `GameplayAbilities` 的具体用法），可以继续探讨！

### 补充部分模块
我发现它漏了几个模块的介绍,所以我又单独问了下它.
PropertyPath,RHI,Gauntlet,ClientPilot

感谢你的指正！以下是补充分析的 **`PropertyPath`、`RHI`、`Gauntlet`、`ClientPilot`** 四个模块的详细作用，结合它们在 `LyraGame` 中的使用场景：

---

#### 1. PropertyPath
- **模块类型**：公有依赖（`PublicDependencyModuleNames`）
- **作用**：  
  - **属性路径解析**：提供动态访问和修改对象属性（`UObject` 成员变量）的能力，支持嵌套属性（如 `Actor.Component.Value`）。  
  - **使用场景**：  
    - 数据驱动逻辑（如通过字符串路径动态修改技能参数）。  
    - 编辑器工具链（如自动化测试时批量调整属性）。  
- **Lyra中的用途**：  
  - 可能用于动态调整角色属性（如伤害计算、技能效果）或游戏配置（如难度参数）。

---

#### 2. RHI (Rendering Hardware Interface)
- **模块类型**：私有依赖（`PrivateDependencyModuleNames`）
- **作用**：  
  - **底层图形API抽象层**：封装了 DirectX/Vulkan/Metal 等图形接口，提供跨平台的渲染指令。  
  - **核心功能**：  
    - 管理纹理、缓冲区、着色器等GPU资源。  
    - 处理多线程渲染命令提交。  
- **Lyra中的用途**：  
  - 自定义渲染效果（如后处理、技能特效）或性能优化（如异步资源加载）。  
  - 依赖 `RenderCore`（同为私有依赖）实现高级渲染逻辑。

---

#### 3. Gauntlet
- **模块类型**：私有依赖（`PrivateDependencyModuleNames`）
- **作用**：  
  - **自动化测试框架**：用于编写和运行游戏功能测试、压力测试、性能测试等。  
  - **核心功能**：  
    - 模拟玩家输入（如按键、鼠标操作）。  
    - 集成CI/CD流程（如Jenkins中的自动化测试）。  
- **Lyra中的用途**：  
  - 确保核心玩法（如技能系统、网络同步）的稳定性。  
  - 可能用于测试战斗平衡性、多人同步问题等。

---

#### 4. ClientPilot
- **模块类型**：私有依赖（`PrivateDependencyModuleNames`）
- **作用**：  
  - **客户端自动化行为控制**：模拟玩家操作（如挂机、脚本行为），用于测试或演示。  
  - **与Gauntlet的区别**：  
    - `Gauntlet` 是全局测试框架，`ClientPilot` 更侧重单个客户端的自动化行为。  
  - **典型用例**：  
    - 录制和回放玩家操作。  
    - 生成AI控制的“机器人”玩家。  
- **Lyra中的用途**：  
  - 可能用于多人模式的机器人填充（如未满员时自动补位）。  
  - 开发期快速验证游戏流程（如自动完成关卡）。

---

#### 总结：这些模块如何服务LyraGame**
| 模块           | 分类       | 对Lyra的核心贡献                                  |
|----------------|------------|--------------------------------------------------|
| **PropertyPath** | 公有依赖   | 动态属性调整，支持数据驱动设计。                  |
| **RHI**         | 私有依赖   | 底层渲染控制，保障跨平台图形效果。                |
| **Gauntlet**    | 私有依赖   | 自动化测试，确保玩法系统稳定性。                  |
| **ClientPilot** | 私有依赖   | 模拟玩家行为，辅助开发和测试。                    |

---

#### 关联性问题思考
1. **为什么 `RHI` 是私有依赖？**  
   - 渲染实现是模块内部细节，不应暴露给其他模块，避免耦合。

2. **`Gauntlet` 和 `ClientPilot` 如何协作？**  
   - `Gauntlet` 控制全局测试流程，`ClientPilot` 实现单个客户端的自动化行为，二者可结合用于复杂场景测试（如100个机器人压力测试）。

3. **`PropertyPath` 的替代方案？**  
   - 手动编写属性访问代码，但会失去灵活性和可配置性。

如果需要进一步探讨某个模块的具体实现（如如何在Lyra中使用 `ClientPilot` 生成机器人），可以继续深入！

## 补充一下路径检索
这里补充一下个人习惯,有时候为了偷懒.实在不想写路径.我会将该模块下的所有文件夹都纳入包含路径.
这样我在引入路径的时候,只需要写头文件即可.
当然,这是我在VS2022中的做法.Rider好像已经支持自动管理头文件了.......
但是手动也没什么问题哈!
``` cs
// Copyright 2023 Xiao Gang. All Rights Reserved.

using UnrealBuildTool;

public class XGXunFeiCore : ModuleRules
{
    public XGXunFeiCore(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = ModuleRules.PCHUsageMode.UseExplicitOrSharedPCHs;

        PublicIncludePaths.AddRange(
            new string[] {
             "$(ModuleDir)/Public/Async",
             "$(ModuleDir)/Public/Config",
            }
            );


        PrivateIncludePaths.AddRange(
            new string[] {
               "XGXunFeiCore/Public/Async",
               "XGXunFeiCore/Public/Config",
               "XGXunFeiCore/Public/Log",
               "XGXunFeiCore/Public/Type"
            }
            );


        PublicDependencyModuleNames.AddRange(
            new string[]
            {
                "Core",
                "Json",
                "JsonUtilities",

            }
            );


        PrivateDependencyModuleNames.AddRange(
            new string[]
            {
                "CoreUObject",
                "Engine",
                "DeveloperSettings",
                "XGXunFeiBase",
                "Projects",
                "ImageWrapper"
            }
            );

        //Need to make sure Android has Launch module.
        if (Target.Platform == UnrealTargetPlatform.Android)
        {
            PrivateIncludePathModuleNames.AddRange(
                new string[] {
                    "Launch"
                }
            );
        }




    }
}

```

## 总结

本节主要是完善了两个Build文件,把我们所需要用到的插件代码导入其中,并将对应的插件和模块依赖也导入好.
值得注意的是Lyra是EPIC推出的一个虚幻引擎官方示例,里面涵盖了各种中高级功能开发示例,还在不断持续集成开发中.并非所有功能都是正式稳定的发行版本.
偷偷地说一句,感觉大模型是不是已经吃过了虚幻引擎相关引擎和工程源码.感觉回答的很有精神!
