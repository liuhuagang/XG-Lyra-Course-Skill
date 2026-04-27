# UE5_Lyra学习指南_004_创建工程

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

- [UE5\_Lyra学习指南\_004\_创建工程](#ue5_lyra学习指南_004_创建工程)
  - [创建UE5.6新工程](#创建ue56新工程)
  - [添加Editor模块](#添加editor模块)
  - [阐述cs配置文件](#阐述cs配置文件)
    - [1. `.target.cs` 文件（定义编译目标）](#1-targetcs-文件定义编译目标)
    - [2. `.build.cs` 文件（定义模块构建规则）](#2-buildcs-文件定义模块构建规则)
    - [关键区别](#关键区别)
    - [典型工作流程](#典型工作流程)
    - [注意事项](#注意事项)
  - [总结](#总结)
  - [参考文献](#参考文献)




## 创建UE5.6新工程

我们需要创建一个空白的UEC++工程.不包含任何模板.

通过Epic客户端创建即可

## 添加Editor模块

Lyra示例工程有两个模块,分别是LyraGame游戏模块和LyraEditor编辑器模块.

Epic客户端创建的Lyra工程只有一个Lyra模块.不能满足我们的要求.我们需要拷贝一份该模块代码,并对这两个模块进行重命名即可.

此处需要注意,任何改动不匹配都会导致项目编译或者刷新失败!!

此处不赘述什么是模块,创建一个模块需要创建哪些文件.

如果有小伙伴不太清楚模块的基本知识,请先学习小刚的基础课程《[UE5]虚幻引擎UEC++从基础到进阶》

模块是虚幻引擎（UE）的构建块。 引擎本质上就是众多模块组成的大规模集合，游戏则提供自己的模块来扩充这个集合。 每个模块都封装了一组功能，并且可以提供公共接口和编译环境（包括宏、路径等）供其他模块使用。

模块通过C#源文件进行声明，这些文件的扩展名为.build.cs，存储在你的项目的Source目录下。 属于某个模块的C++源代码与.build.cs文件并列存储，或者存储在它的子目录中。 每个.build.cs文件都会声明一个衍生于ModuleRules基类的类，并设置属性来控制应如何通过其构造函数进行编译。 这些.build.cs文件由虚幻编译工具（UBT）编译并构造，同时决定了整体编译环境。

.build.cs 文件的典型结构如下。

```
using UnrealBuildTool;
	using System.Collections.Generic;
	public class MyModule : ModuleRules
	{
		public MyModule(ReadOnlyTargetRules Target) : base(Target)
		{
			// Properties and settings go here
		}
	}

```



在创建完模块后,我们需要在.uplugin或.uproject中指定模块的加载类型和时机.否则该模块不能正常识别使用.

此处贴出本节Lyra.uproject:
``` json
{
	"FileVersion": 3,
	"EngineAssociation": "5.6",
	"Category": "",
	"Description": "",
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
	]
}
```
## 阐述cs配置文件

此处贴出DeepSeek回答仅供参考如下:

在 **Unreal Engine (虚幻引擎)** 的 C++ 项目开发中，`.target.cs` 和 `.build.cs` 是两种重要的配置文件，用于控制项目的 **编译目标（Target）** 和 **模块构建规则（Build）**。它们的作用如下：

### 1. `.target.cs` 文件（定义编译目标）
- **作用**：  
  用于定义 **可执行目标**（如游戏、编辑器工具、服务器等），指定如何编译生成最终的可执行文件（`.exe` 或平台对应的二进制文件）。
- **文件名格式**：  
  `[项目名].Target.cs`（如 `MyGame.Target.cs`）或 `[项目名]Editor.Target.cs`（如 `MyGameEditor.Target.cs`）。
- **常见用途**：
  - 定义目标类型（游戏、编辑器、客户端、服务器等）。
  - 配置依赖模块（如 `Core`, `Engine`, `InputCore` 等）。
  - 设置平台特定的编译选项（如是否启用 `Debug` 符号）。
  - 控制是否包含 `Unreal Editor` 相关功能（如 `EditorTarget`）。
- **示例代码**：
  ```csharp
  using UnrealBuildTool;

  public class MyGameTarget : TargetRules
  {
      public MyGameTarget(TargetInfo Target) : base(Target)
      {
          Type = TargetType.Game; // 目标类型（游戏、编辑器等）
          DefaultBuildSettings = BuildSettingsVersion.V2;
          ExtraModuleNames.Add("MyGame"); // 依赖的主模块
      }
  }
  ```
- **常见类型**：
  - `TargetType.Game`：游戏客户端。
  - `TargetType.Editor`：编辑器扩展。
  - `TargetType.Server`：专用服务器。

---

### 2. `.build.cs` 文件（定义模块构建规则）
- **作用**：  
  用于定义 **模块（Module）** 的构建规则，指定模块的依赖项、编译选项、库文件等。
- **文件名格式**：  
  `[模块名].Build.cs`（如 `MyGame.Build.cs`）。
- **常见用途**：
  - 声明模块的公共/私有依赖（如 `CoreUObject`, `Slate`, `AIModule`）。
  - 添加第三方库（如 `.lib`, `.dll`）。
  - 配置预处理器宏（如 `WITH_EDITOR`）。
  - 控制模块是否支持特定平台（如 `Windows`, `Android`）。
- **示例代码**：
  ```csharp
  using UnrealBuildTool;

  public class MyGame : ModuleRules
  {
      public MyGame(ReadOnlyTargetRules Target) : base(Target)
      {
          PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

          PublicDependencyModuleNames.AddRange(new string[] { 
              "Core", 
              "CoreUObject", 
              "Engine", 
              "InputCore" 
          });

          PrivateDependencyModuleNames.AddRange(new string[] { 
              "Slate", 
              "AIModule" 
          });

          // 添加第三方库
          if (Target.Platform == UnrealTargetPlatform.Win64)
          {
              PublicAdditionalLibraries.Add("ThirdParty.lib");
          }
      }
  }
  ```

---

### 关键区别
| 文件类型       | 作用范围           | 控制内容                     | 示例文件名         |
|----------------|--------------------|------------------------------|--------------------|
| `.Target.cs`   | 整个项目的编译目标 | 定义生成的可执行文件类型     | `MyGame.Target.cs` |
| `.Build.cs`    | 单个模块的构建规则 | 配置模块的依赖和编译选项     | `MyGame.Build.cs`  |

---

### 典型工作流程
1. **`.build.cs`**：  
   模块开发者通过它声明模块的依赖和编译规则。例如，如果你的模块需要用到 `UMG`，就在 `PublicDependencyModuleNames` 中添加 `"UMG"`。
2. **`.target.cs`**：  
   项目构建时，引擎根据 `.Target.cs` 决定如何链接模块。例如，`EditorTarget` 会包含编辑器专用的模块（如 `UnrealEd`）。

---

### 注意事项
- 修改 `.build.cs` 或 `.Target.cs` 后，需要 **重新生成项目文件**（右键 `.uproject` → **Generate Visual Studio Project Files**）。
- 避免在 `.build.cs` 中直接硬编码平台路径，推荐使用 `UE` 提供的工具链（如 `ThirdParty` 目录管理）。

如果需要更复杂的配置（如自定义编译步骤），可以进一步研究 `UnrealBuildTool` 的 API 文档。
## 总结

本节构建了名为Lyra工程的基本架构,创建了两个模块,除了模块的CPP文件,没有新增任何CPP文件.
并简单复习了.Target.cs和Build.cs文件.
此时工程可以编译成功,并顺利启动.

## 参考文献
[模块属性](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/module-properties-in-unreal-engine)