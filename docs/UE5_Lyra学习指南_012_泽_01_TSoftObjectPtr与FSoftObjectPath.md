# UE5_Lyra学习指南_012_泽_01_TSoftObjectPtr与FSoftObjectPath

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)

文章内容由小刚撰写,采用了以下多种方式:

1.口述转文字
2.AI重构
3.参考引擎源码
4.Lyra工程源码
5.结合社区论坛各位大佬的解析

- [UE5\_Lyra学习指南\_012\_泽\_01\_TSoftObjectPtr与FSoftObjectPath](#ue5_lyra学习指南_012_泽_01_tsoftobjectptr与fsoftobjectpath)
  - [概述](#概述)
    - [1. **`FSoftObjectPath`**](#1-fsoftobjectpath)
    - [2. **`TSoftObjectPtr<T>`**](#2-tsoftobjectptrt)
    - [3. **`FSoftObjectPtr`**](#3-fsoftobjectptr)
    - [三者的关系总结](#三者的关系总结)
    - [额外说明](#额外说明)
  - [总结](#总结)



## 概述

UE5中,如何理解TSoftObjectPtr,FSoftObjectPath,FSoftObjectPtr这几者之间的关系

在Unreal Engine 5中，`TSoftObjectPtr`、`FSoftObjectPath`和`FSoftObjectPtr`都是用于处理软引用（Soft Reference）的类，它们之间的关系和区别如下：

---

### 1. **`FSoftObjectPath`**
   - **本质**：存储一个资源的**软引用路径**（例如`/Game/Path/To/Asset.AssetName`）。
   - **用途**：
     - 仅保存资源的路径字符串，不加载资源。
     - 用于序列化（保存到磁盘）或传递资源路径。
   - **特点**：
     - 轻量级，不涉及资源加载或内存管理。
     - 可以通过`TryLoad()`或`ResolveObject()`手动加载资源。
   - **示例**：
     ```cpp
     FSoftObjectPath Path("/Game/Characters/Player.Player");
     UObject* LoadedObject = Path.TryLoad(); // 手动加载
     ```

---

### 2. **`TSoftObjectPtr<T>`**
   - **本质**：`FSoftObjectPath`的**模板化封装**，提供类型安全的软引用。
   - **用途**：
     - 存储对特定类型（如`UBlueprint`、`UTexture`）的软引用。
     - 自动处理路径解析和类型检查。
   - **特点**：
     - 内部包含`FSoftObjectPath`，但通过模板参数`T`限定资源类型。
     - 提供`Get()`、`LoadSynchronous()`等便捷方法。
     - 适合在代码中声明对特定类型资源的软引用。
   - **示例**：
     ```cpp
     TSoftObjectPtr<UTexture2D> SoftTexturePtr("/Game/Textures/Icon.Icon");
     UTexture2D* Texture = SoftTexturePtr.LoadSynchronous(); // 同步加载
     ```

---

### 3. **`FSoftObjectPtr`**
   - **本质**：`TSoftObjectPtr<UObject>`的**非模板化基类**。
   - **用途**：
     - 存储对任意`UObject`的软引用，无需指定具体类型。
     - 适用于需要动态处理不同类型资源的场景。
   - **特点**：
     - 类似`TSoftObjectPtr`，但失去类型安全（返回的是`UObject*`）。
     - 通常用于底层系统或非模板代码。
   - **示例**：
     ```cpp
     FSoftObjectPtr SoftPtr("/Game/Assets/AnyObject.AnyObject");
     UObject* Obj = SoftPtr.Get(); // 返回基类UObject
     ```

``` txt
/**
* FSoftObjectPtr 是一种指向 UObject 的弱指针类型，它还会记录该对象在磁盘上的路径。
* 当所引用的对象被加载或卸载时，它会不断在“有效”和“等待”状态之间转换。
* 它不会影响对象是否会被垃圾回收。*
* 这对于明确那些您可能希望按需异步加载的资产很有帮助。
*/


```
---

### 三者的关系总结
| 类名                  | 类型安全 | 内部存储          | 典型用途                          |
|-----------------------|----------|-------------------|-----------------------------------|
| `FSoftObjectPath`     | ❌ 无     | 纯路径字符串      | 序列化、动态资源路径处理          |
| `TSoftObjectPtr<T>`   | ✅ 有     | `FSoftObjectPath` | 类型安全的资源软引用（如`UTexture`） |
| `FSoftObjectPtr`      | ❌ 无     | `FSoftObjectPath` | 非模板化的通用软引用              |

- **继承关系**：  
  `TSoftObjectPtr<T>` 是模板类，`FSoftObjectPtr` 是其非模板化的特例（`TSoftObjectPtr<UObject>`的别名）。  
  `FSoftObjectPath` 是两者底层路径存储的核心。

- **选择建议**：  
  - 需要类型安全 → **`TSoftObjectPtr<T>`**  
  - 需要动态类型或底层操作 → **`FSoftObjectPath`/`FSoftObjectPtr`**  
  - 需要序列化路径 → **`FSoftObjectPath`**

---

### 额外说明
- **异步加载**：以上类通常配合`FStreamableManager`或`AsyncLoading`实现异步加载。
- **编辑器兼容性**：在编辑器中，这些类会跟踪资源移动/重命名，自动更新路径。

通过理解这些差异，可以更高效地管理UE5中的资源引用！

## 总结
以上部分内容参考DeepSeek.