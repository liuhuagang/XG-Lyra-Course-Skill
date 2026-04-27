# UE5_Lyra学习指南_009_Engine类

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
- [UE5\_Lyra学习指南\_009\_Engine类](#ue5_lyra学习指南_009_engine类)
- [UE5\_Lyra学习指南-009 Engine类](#ue5_lyra学习指南-009-engine类)
  - [引擎类继承关系](#引擎类继承关系)
  - [引擎加载时机](#引擎加载时机)
  - [引擎类配置与初始化](#引擎类配置与初始化)
    - [1. 配置来源](#1-配置来源)
    - [2. 引擎类区别](#2-引擎类区别)
  - [验证当前引擎类](#验证当前引擎类)
    - [方法1：控制台命令](#方法1控制台命令)
    - [方法2：查看启动日志](#方法2查看启动日志)
    - [方法3：代码调试](#方法3代码调试)
  - [自定义引擎类示例](#自定义引擎类示例)
    - [1. 创建派生类](#1-创建派生类)
    - [2. 修改配置](#2-修改配置)
    - [3. 重新编译](#3-重新编译)
  - [关键流程总结](#关键流程总结)
  - [修正说明](#修正说明)

---

# UE5_Lyra学习指南-009 Engine类

## 引擎类继承关系
```cpp
UEngine (基类)
├─ UGameEngine (游戏运行时引擎)
└─ UEditorEngine (通用编辑器引擎基类)
   └─ UUnrealEdEngine (虚幻专用编辑器引擎)
```
引擎类位置
E:\Epic\UE\UE_5.6\Engine\Source\Runtime\Engine\Classes\Engine\Engine.h
``` cpp
/** Sets the class to use for the game viewport client, which can be overridden to change game-specific input and display behavior. */
/** 设置用于游戏视口客户端的类，该类可被重写以改变游戏特定的输入和显示行为。*/
UPROPERTY(globalconfig, noclear, EditAnywhere, Category=DefaultClasses, meta=(MetaClass="/Script/Engine.GameViewportClient", DisplayName="Game Viewport Client Class",ConfigRestartRequired=true))
FSoftClassPath GameViewportClientClassName;

/** Sets the class to spawn as the global AssetManager, configurable per game. If empty, it will not spawn one */
/** 设置用于生成资源的类为全局的 AssetManager 类，该设置可在每个游戏中进行配置。若为空，则不会生成此类资源 */
UPROPERTY(globalconfig, noclear, EditAnywhere, Category=DefaultClasses, meta=(MetaClass="/Script/CoreUObject.Object", DisplayName="Asset Manager Class", ConfigRestartRequired=true), AdvancedDisplay)
FSoftClassPath AssetManagerClassName;

/** Sets the class to use for WorldSettings, which can be overridden to store game-specific information on map/world. */
UPROPERTY(globalconfig, noclear, EditAnywhere, Category=DefaultClasses, meta=(MetaClass="/Script/Engine.WorldSettings", DisplayName="World Settings Class", ConfigRestartRequired=true))
FSoftClassPath WorldSettingsClassName;


/** Sets the class to use for local players, which can be overridden to store game-specific information for a local player. */
/** 设置用于本地玩家的类，该类可被重写以存储针对本地玩家的特定游戏信息。*/
UPROPERTY(globalconfig, noclear, EditAnywhere, Category=DefaultClasses, meta=(MetaClass="/Script/Engine.LocalPlayer", DisplayName="Local Player Class", ConfigRestartRequired=true))
FSoftClassPath LocalPlayerClassName;


/** Sets the GameUserSettings class, which can be overridden to support game-specific options for Graphics/Sound/Gameplay. */
/** 设置 GameUserSettings 类，该类可被重写以支持与游戏相关的特定图形、声音和游戏玩法选项。*/
UPROPERTY(globalconfig, noclear, EditAnywhere, Category=DefaultClasses, meta=(MetaClass="/Script/Engine.GameUserSettings", DisplayName="Game User Settings Class", ConfigRestartRequired=true), AdvancedDisplay)
FSoftClassPath GameUserSettingsClassName;

```

对应了Lyra的项目设置
``` ini
[/Script/Engine.Engine]
DurationOfErrorsAndWarningsOnHUD=3.0
GameEngine=/Script/LyraGame.LyraGameEngine
UnrealEdEngine=/Script/LyraEditor.LyraEditorEngine
EditorEngine=/Script/LyraEditor.LyraEditorEngine
GameViewportClientClassName=/Script/LyraGame.LyraGameViewportClient
AssetManagerClassName=/Script/LyraGame.LyraAssetManager
WorldSettingsClassName=/Script/LyraGame.LyraWorldSettings
LocalPlayerClassName=/Script/LyraGame.LyraLocalPlayer
GameUserSettingsClassName=/Script/LyraGame.LyraSettingsLocal
NearClipPlane=3.000000
```

## 引擎加载时机
引擎初始化发生在极早期阶段（`main()`之前），核心代码位于：  
`Engine/Source/Runtime/Launch/Private/LaunchEngineLoop.cpp` (约4530行)  
**调试建议**：  
1. 使用源码版本引擎或下载调试符号  
2. 在`FEngineLoop::PreInit()`打断点观察流程  

关键代码段：
```cpp
if(!GIsEditor) { // 游戏模式
    GConfig->GetString(TEXT("/Script/Engine.Engine"), TEXT("GameEngine"), GameEngineClassName, GEngineIni);
    EngineClass = StaticLoadClass(UGameEngine::StaticClass(), nullptr, *GameEngineClassName);
    GEngine = NewObject<UEngine>(GetTransientPackage(), EngineClass);
} 
else { // 编辑器模式
    GConfig->GetString(TEXT("/Script/Engine.Engine"), TEXT("UnrealEdEngine"), UnrealEdEngineClassName, GEngineIni);
    EngineClass = StaticLoadClass(UUnrealEdEngine::StaticClass(), nullptr, *UnrealEdEngineClassName);
    GEngine = GEditor = GUnrealEd = NewObject<UUnrealEdEngine>(GetTransientPackage(), EngineClass);
}
```

---

## 引擎类配置与初始化

### 1. 配置来源
| 配置文件            | 作用                                                                 |
|---------------------|----------------------------------------------------------------------|
| `BaseEngine.ini`    | 引擎默认配置（不可修改）                                             |
| `DefaultEngine.ini` | 项目级配置（覆盖基类配置）                                           |
| 平台特定INI         | 如`Windows/Android`目录下的配置                                      |
| `Saved/Config/...`  | 用户运行时覆盖配置                                                   |

**关键配置项**：
```ini
[/Script/Engine.Engine]
GameEngine=/Script/Engine.GameEngine       ; 游戏运行时引擎类
EditorEngine=/Script/UnrealEd.UnrealEdEngine ; 编辑器运行时引擎类
```

---

### 2. 引擎类区别
| 特性                | `UGameEngine`              | `UEditorEngine` (基类)      | `UUnrealEdEngine` (子类)        |
|---------------------|----------------------------|-----------------------------|----------------------------------|
| **用途**            | 纯游戏运行                 | 编辑器基础框架              | 虚幻编辑器完整实现              |
| **模块依赖**        | 仅核心游戏模块             | 基础编辑器工具              | 全部编辑器工具链（如蓝图调试）  |
| **运行场景**        | 打包后的游戏               | 自定义编辑器工具开发        | Unreal Editor                   |
| **是否支持PIE**     | 否                         | 是（通过派生类实现）        | 是（完整PIE/SIE支持）           |

---

## 验证当前引擎类

### 方法1：控制台命令
```bash
obj list class=Engine
```
输出示例：
```
Name: GEngine, Class: UnrealEdEngine  # 编辑器模式
Name: GEngine, Class: GameEngine      # 游戏模式
```

### 方法2：查看启动日志
注:这个地方是AI编的 实际没有打印这个日志
```log
LogInit: Creating Engine: UnrealEdEngine
LogInit: Initializing GameEngine...
```

### 方法3：代码调试
- 断点位置：  
  `UGameEngine::Init()` 或 `UUnrealEdEngine::Init()`

---

## 自定义引擎类示例

### 1. 创建派生类
```cpp
// MyGameEngine.h
#include "Engine/GameEngine.h"
#include "MyGameEngine.generated.h"

UCLASS()
class MYMODULE_API UMyGameEngine : public UGameEngine {
    GENERATED_BODY()
public:
    virtual void Init(IEngineLoop* InEngineLoop) override {
        UE_LOG(LogTemp, Warning, TEXT("Custom Engine Initialized!"));
        Super::Init(InEngineLoop);
    }
};
```

### 2. 修改配置
`DefaultEngine.ini`：
```ini
[/Script/Engine.Engine]
GameEngine=/Script/MyModule.MyGameEngine
```

### 3. 重新编译
确保模块在`MyModule.Build.cs`中正确声明：
```csharp
PublicDependencyModuleNames.AddRange(new[] { "Engine" });
```

---

## 关键流程总结
1. **配置读取**：  
   - 优先加载`DefaultEngine.ini`覆盖`BaseEngine.ini`  
   - 通过`GIsEditor`决定加载`GameEngine`或`UnrealEdEngine`

2. **类实例化**：  
   - 使用`StaticLoadClass`动态加载指定类  
   - 通过`NewObject`创建实例（`GetTransientPackage()`避免持久化）

3. **初始化**：  
   - `UGameEngine`：直接启动游戏逻辑  
   - `UUnrealEdEngine`：加载编辑器子系统（如AssetTools、LevelEditor）

---

## 修正说明
本节简单的介绍了引擎类的部分基础知识.


