# UE5_Lyra学习指南__034_AI_ActorFactory

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_999\_编辑模板](#ue5_lyra学习指南_999_编辑模板)
		- [核心作用：](#核心作用)
		- [工作流程（以拖动静态网格为例）：](#工作流程以拖动静态网格为例)
		- [自定义 ActorFactory：](#自定义-actorfactory)
		- [常见内置 ActorFactory 示例：](#常见内置-actorfactory-示例)
		- [使用场景：](#使用场景)
		- [注意事项：](#注意事项)

## 概述

在 Unreal Engine 5 (UE5) 中，`ActorFactory` 是一个**用于在编辑器中动态生成特定类型 Actor 的工具类**。它充当了资源（如静态网格、蓝图、音效等）与场景中实际 Actor 实例之间的桥梁，简化了从内容浏览器拖放资源到关卡的过程。



### 核心作用：
1. **资源到Actor的转换**  
   当从内容浏览器拖动资源（如 `.uasset` 文件）到关卡视图时，UE5 会自动查找能处理该资源类型的 `ActorFactory`，并生成对应的 Actor。  
   **示例**：  
   - 拖动静态网格（`.uasset`） → 生成 `StaticMeshActor`  
   - 拖动蓝图类（`.Blueprint`） → 生成该蓝图的实例  
   - 拖动音效（`.SoundWave`） → 生成 `AmbientSoundActor`

2. **自动化配置**  
   `ActorFactory` 在创建 Actor 时会自动完成基础配置：  
   - 将资源关联到 Actor 组件（如为 `StaticMeshActor` 设置网格）  
   - 设置默认属性（如碰撞、物理等）  
   - 应用合理的初始变换（位置/旋转/缩放）

3. **扩展编辑器功能**  
   开发者可自定义 `ActorFactory` 子类，实现特殊资源的拖放生成逻辑（如自定义数据资产生成特定游戏对象）。

---

### 工作流程（以拖动静态网格为例）：
1. 用户从内容浏览器拖动 `MyMesh.uasset` 到关卡。
2. 编辑器遍历所有已注册的 `ActorFactory`，调用 `CanCreateActorFrom()` 检查哪个工厂支持该资源。
3. 找到 `ActorFactoryStaticMesh`（内置工厂），调用其 `CreateActor()` 方法。
4. 工厂生成一个 `StaticMeshActor`，并将 `MyMesh` 赋给其 `UStaticMeshComponent`。
5. 新 Actor 被放置到拖放位置。

---

### 自定义 ActorFactory：
如需为自定义资源类型创建工厂，需继承 `UActorFactory` 并重写关键方法：

```cpp
// 自定义工厂头文件
#pragma once
#include "ActorFactory.h"
#include "MyCustomActorFactory.generated.h"

UCLASS()
class UMyCustomActorFactory : public UActorFactory
{
    GENERATED_BODY()
public:
    UMyCustomActorFactory();
    
    // 检查资源是否支持
    virtual bool CanCreateActorFrom(const FAssetData& AssetData, FText& OutErrorMsg) override;
    
    // 实际生成Actor
    virtual AActor* CreateActor(const FAssetData& AssetData, ULevel* InLevel, const FTransform& Transform, EObjectFlags ObjectFlags, const FName Name) override;
};
```

```cpp
// 实现文件
#include "MyCustomActorFactory.h"
#include "MyCustomAsset.h" // 你的自定义资源类型
#include "MyCustomActor.h" // 要生成的Actor类型

UMyCustomActorFactory::UMyCustomActorFactory()
{
    DisplayName = FText::FromString("My Custom Actor");
    NewActorClass = AMyCustomActor::StaticClass();
}

bool UMyCustomActorFactory::CanCreateActorFrom(const FAssetData& AssetData, FText& OutErrorMsg)
{
    return AssetData.IsInstanceOf(UMyCustomAsset::StaticClass());
}

AActor* UMyCustomActorFactory::CreateActor(...)
{
    AActor* NewActor = Super::CreateActor(...);
    if (AMyCustomActor* CustomActor = Cast<AMyCustomActor>(NewActor))
    {
        UMyCustomAsset* CustomAsset = Cast<UMyCustomAsset>(AssetData.GetAsset());
        CustomActor->InitializeFromAsset(CustomAsset); // 自定义初始化
    }
    return NewActor;
}
```

---

### 常见内置 ActorFactory 示例：
| 资源类型             | 生成的 Actor             | 工厂类                     |
|----------------------|--------------------------|----------------------------|
| 静态网格 (.uasset)   | StaticMeshActor          | `ActorFactoryStaticMesh`   |
| 蓝图类 (.Blueprint)  | 该蓝图的实例             | `ActorFactoryBlueprint`    |
| 音效 (.SoundWave)    | AmbientSoundActor        | `ActorFactoryBasicSound`   |
| 贴花材质 (.Material) | DecalActor               | `ActorFactoryDecal`        |

---

### 使用场景：
- **快速关卡搭建**：通过拖放资源快速填充场景。
- **自定义工具开发**：为插件/工具创建专用的资源放置逻辑。
- **数据驱动生成**：从数据资产（如Excel导入的配置）动态生成游戏对象。

### 注意事项：
- **仅限编辑器**：`ActorFactory` 主要在编辑阶段使用，不影响运行时逻辑。
- **优先级**：多个工厂支持同一资源时，UE5 选择最匹配的（可通过重写 `GetDefaultActorClass()` 调整）。
- **模块依赖**：自定义工厂需在模块的 `StartupModule()` 中注册（通常自动完成）。

通过 `ActorFactory`，UE5 实现了资源与场景对象的无缝衔接，极大提升了关卡设计效率。

## 总结
以上回答来自DeepSeek.