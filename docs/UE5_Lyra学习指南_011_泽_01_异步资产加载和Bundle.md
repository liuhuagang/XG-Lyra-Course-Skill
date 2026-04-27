# UE5_Lyra学习指南_011_泽_01_异步资产加载和Bundle

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)

文章内容由小刚撰写,采用了以下多种方式:

1.口述转文字
2.AI重构
3.参考引擎源码
4.Lyra工程源码
5.结合社区论坛各位大佬的解析

- [UE5\_Lyra学习指南\_011\_泽\_01\_异步资产加载和Bundle](#ue5_lyra学习指南_011_泽_01_异步资产加载和bundle)
- [ChangeBundleStateForPrimaryAssets](#changebundlestateforprimaryassets)
- [理解 UE5 中的 `ChangeBundleStateForPrimaryAssets` 函数](#理解-ue5-中的-changebundlestateforprimaryassets-函数)
  - [函数参数解析](#函数参数解析)
  - [Bundles 概念详解](#bundles-概念详解)
  - [典型使用场景](#典型使用场景)
  - [内部工作原理](#内部工作原理)
  - [最佳实践](#最佳实践)
- [LoadAssetList](#loadassetlist)
- [理解 UE5 中的 `LoadAssetList` 函数](#理解-ue5-中的-loadassetlist-函数)
  - [函数签名解析](#函数签名解析)
  - [参数详解](#参数详解)
  - [返回值](#返回值)
  - [核心功能理解](#核心功能理解)
  - [使用示例](#使用示例)
    - [基本用法](#基本用法)
    - [带句柄的高级用法](#带句柄的高级用法)
  - [与 ChangeBundleStateForPrimaryAssets 的区别](#与-changebundlestateforprimaryassets-的区别)
  - [最佳实践](#最佳实践-1)

# ChangeBundleStateForPrimaryAssets
``` txt
在UE5加载资产过程中,如何理解virtual TSharedPtr<FStreamableHandle> ChangeBundleStateForPrimaryAssets(
		const TArray<FPrimaryAssetId>& AssetsToChange,
		const TArray<FName>& AddBundles,
		const TArray<FName>& RemoveBundles,
		bool bRemoveAllBundles = false,
		FStreamableDelegate DelegateToCall = FStreamableDelegate(),
		TAsyncLoadPriority Priority = FStreamableManager::DefaultAsyncLoadPriority,
		UE::FSourceLocation Location = UE::FSourceLocation::Current());这个函数 特别是其中的关于Bundles的含义
```
# 理解 UE5 中的 `ChangeBundleStateForPrimaryAssets` 函数

`ChangeBundleStateForPrimaryAssets` 是 Unreal Engine 5 中用于管理主资产(primary assets)加载状态的函数，特别是控制它们的"bundles"(包)状态。让我们分解这个函数并重点理解"bundles"的概念。

## 函数参数解析

```cpp
virtual TSharedPtr<FStreamableHandle> ChangeBundleStateForPrimaryAssets(
    const TArray<FPrimaryAssetId>& AssetsToChange,
    const TArray<FName>& AddBundles,
    const TArray<FName>& RemoveBundles,
    bool bRemoveAllBundles = false,
    FStreamableDelegate DelegateToCall = FStreamableDelegate(),
    TAsyncLoadPriority Priority = FStreamableManager::DefaultAsyncLoadPriority,
    UE::FSourceLocation Location = UE::FSourceLocation::Current());
```

- **AssetsToChange**: 要修改的主资产ID列表
- **AddBundles**: 要添加到这些资产的bundle列表
- **RemoveBundles**: 要从这些资产中移除的bundle列表
- **bRemoveAllBundles**: 是否移除所有bundle(忽略RemoveBundles参数)
- **DelegateToCall**: 操作完成时调用的委托
- **Priority**: 异步加载的优先级
- **Location**: 调试用的源代码位置信息

## Bundles 概念详解

在UE5中，**bundles(包)** 是资产管理系统中的一个核心概念，它们代表资产的特定使用状态或用途。理解bundles的关键点：

1. **动态加载控制**：Bundles允许你精细控制何时加载资产的特定部分。例如，一个角色资产可能有"MainGame"和"Cutscene"两个bundle，包含不同的LOD或纹理质量。

2. **内存管理**：通过激活/停用特定bundles，你可以确保只加载当前需要的资产部分，减少内存占用。

3. **多用途资产**：单个资产可以服务于多个用途，每个用途对应不同的bundle。

## 典型使用场景

1. **按需加载**：
   ```cpp
   // 加载主角色资产的高清纹理(仅当进入摄影模式时)
   ChangeBundleStateForPrimaryAssets({CharacterId}, {"Photomode"}, {}, false);
   ```

2. **场景切换**：
   ```cpp
   // 从主菜单切换到游戏场景时
   ChangeBundleStateForPrimaryAssets(
       LevelAssets, 
       {"Gameplay"}, 
       {"MainMenu"}, 
       false,
       OnLevelLoadedDelegate);
   ```

3. **内存优化**：
   ```cpp
   // 卸载所有不再需要的bundles
   ChangeBundleStateForPrimaryAssets(OldAssets, {}, {}, true);
   ```

## 内部工作原理

当调用此函数时，引擎会：

1. 更新指定资产的bundle状态
2. 根据新的bundle状态决定哪些部分需要加载或卸载
3. 异步执行必要的加载/卸载操作
4. 完成后调用委托

## 最佳实践

1. **合理规划bundles**：在设计资产时预先考虑不同的使用场景，定义清晰的bundle策略。

2. **避免过度细分**：太多小的bundles会增加管理复杂度。

3. **生命周期管理**：确保在不再需要时移除bundles，特别是内存密集型资源。

4. **错误处理**：检查返回的`FStreamableHandle`以监控加载状态。

Bundles系统是UE5强大的资产管理工具，合理使用可以显著提高游戏的内存使用效率和加载性能。

# LoadAssetList
``` txt
如何理解这个函数呢	TSharedPtr<FStreamableHandle> LoadAssetList(
		const TArray<FSoftObjectPath>& AssetList,
		FStreamableDelegate DelegateToCall = FStreamableDelegate(),
		TAsyncLoadPriority Priority = FStreamableManager::DefaultAsyncLoadPriority,
		FString&& DebugName = FString(TEXT("LoadAssetList")),
		UE::FSourceLocation&& Location = UE::FSourceLocation::Current());
```
# 理解 UE5 中的 `LoadAssetList` 函数

`LoadAssetList` 是 Unreal Engine 5 中用于异步加载一组资产的函数，属于 `FStreamableManager` 类的核心功能之一。这个函数提供了一种高效的方式来批量加载资源。

## 函数签名解析

```cpp
TSharedPtr<FStreamableHandle> LoadAssetList(
    const TArray<FSoftObjectPath>& AssetList,
    FStreamableDelegate DelegateToCall = FStreamableDelegate(),
    TAsyncLoadPriority Priority = FStreamableManager::DefaultAsyncLoadPriority,
    FString&& DebugName = FString(TEXT("LoadAssetList")),
    UE::FSourceLocation&& Location = UE::FSourceLocation::Current());
```

## 参数详解

1. **`AssetList`**  
   - 类型: `const TArray<FSoftObjectPath>&`
   - 要加载的资产路径列表，使用 `FSoftObjectPath` 引用资产
   - 示例: `/Game/Characters/Hero/Mesh.Hero_Mesh`

2. **`DelegateToCall`**  
   - 类型: `FStreamableDelegate`
   - 当所有资产加载完成时调用的委托
   - 默认是空委托

3. **`Priority`**  
   - 类型: `TAsyncLoadPriority`
   - 异步加载的优先级
   - 默认值是 `FStreamableManager::DefaultAsyncLoadPriority`

4. **`DebugName`**  
   - 类型: `FString&&`
   - 用于调试的标识名称
   - 默认值是 "LoadAssetList"

5. **`Location`**  
   - 类型: `UE::FSourceLocation&&`
   - 调用位置的源代码信息，用于调试
   - 默认是当前调用位置

## 返回值

- 返回 `TSharedPtr<FStreamableHandle>`  
  - 一个共享指针，指向加载操作的句柄
  - 可用于检查加载状态、取消加载或绑定完成后的回调

## 核心功能理解

1. **异步批量加载**  
   函数会异步加载 `AssetList` 中的所有资产，不会阻塞游戏线程。

2. **资源管理**  
   返回的 `FStreamableHandle` 会保持对加载资产的引用，防止被垃圾回收。

3. **回调机制**  
   通过 `DelegateToCall` 可以在加载完成后执行特定逻辑。

## 使用示例

### 基本用法
```cpp
// 创建要加载的资产列表
TArray<FSoftObjectPath> AssetsToLoad;
AssetsToLoad.Add(FSoftObjectPath(TEXT("/Game/Weapons/Sword.Sword")));
AssetsToLoad.Add(FSoftObjectPath(TEXT("/Game/Characters/Hero/Hero.Hero")));

// 加载资产并在完成后打印消息
StreamableManager.LoadAssetList(AssetsToLoad, FStreamableDelegate::CreateLambda([]()
{
    UE_LOG(LogTemp, Display, TEXT("Assets loaded!"));
}));
```

### 带句柄的高级用法
```cpp
// 获取加载句柄
TSharedPtr<FStreamableHandle> Handle = StreamableManager.LoadAssetList(AssetsToLoad);

// 检查是否加载完成
if(Handle.IsValid() && Handle->HasLoadCompleted())
{
    // 使用已加载的资产
}

// 也可以绑定回调到已有句柄
Handle->BindCompleteDelegate(FStreamableDelegate::CreateLambda([]()
{
    // 加载完成后的操作
}));
```

## 与 ChangeBundleStateForPrimaryAssets 的区别

1. **操作对象不同**  
   - `LoadAssetList`: 直接操作资产路径
   - `ChangeBundleStateForPrimaryAssets`: 操作主资产ID和bundles

2. **功能侧重点不同**  
   - `LoadAssetList`: 简单的资产加载
   - `ChangeBundleStateForPrimaryAssets`: 更复杂的bundle状态管理

3. **使用场景不同**  
   - `LoadAssetList`: 当你知道确切需要加载哪些资产时
   - `ChangeBundleStateForPrimaryAssets`: 当需要基于游戏状态动态加载资产时

## 最佳实践

1. **合理使用异步加载**：避免在每帧都创建大量加载请求。

2. **管理加载句柄**：存储返回的句柄以确保资产不被过早释放。

3. **错误处理**：检查资产是否成功加载后再使用。

4. **性能考虑**：根据情况调整加载优先级，重要资源使用更高优先级。

这个函数是UE5异步资源加载系统的核心部分，合理使用可以显著提高游戏的流式加载性能和内存使用效率。

