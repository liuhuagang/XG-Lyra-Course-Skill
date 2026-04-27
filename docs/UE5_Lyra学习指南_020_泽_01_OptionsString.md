# UE5_Lyra学习指南_020_泽_01_OptionsString

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)

文章内容由小刚撰写,采用了以下多种方式:

1.口述转文字
2.AI重构
3.参考引擎源码
4.Lyra工程源码
5.结合社区论坛各位大佬的解析

- [UE5\_Lyra学习指南\_020\_泽\_01\_OptionsString](#ue5_lyra学习指南_020_泽_01_optionsstring)
	- [概述](#概述)
		- [基本作用：](#基本作用)
		- [典型格式：](#典型格式)
		- [常用解析方法：](#常用解析方法)
		- [典型使用场景：](#典型使用场景)
		- [注意事项：](#注意事项)
	- [总结](#总结)



## 概述
在UE5的`AGameModeBase`类中，`OptionsString`是一个用于存储和解析游戏选项的字符串变量。让我详细解释一下它的作用和用法：

### 基本作用：
1. **存储启动参数**：`OptionsString`保存了游戏启动时传入的选项字符串（通常来自命令行参数或URL查询字符串）。
2. **蓝图访问**：由于标记了`BlueprintReadOnly`，你可以在蓝图中读取这个值（但不能修改）。
3. **游戏模式配置**：常用于在游戏开始时根据参数配置游戏规则、地图、玩家数量等。

### 典型格式：
选项字符串通常采用以下格式：
```
?Option1=Value1&Option2=Value2
```
例如：
```
?Difficulty=Hard&EnemyCount=20
```

### 常用解析方法：
虽然`OptionsString`是原始字符串，但通常会使用以下方法解析：

1. **在C++中**：
```cpp
FString Value = UGameplayStatics::ParseOption(OptionsString, "YourOptionName");
```

2. **在蓝图中**：
- 使用"Parse Option"节点（来自`GameplayStatics`）
- 或使用"Split String"等字符串操作节点手动解析

### 典型使用场景：
1. 服务器接收旅行URL中的参数
2. 从主菜单传递设置到游戏场景
3. 调试时快速修改游戏参数

### 注意事项：
- 这个值通常在`InitializeHooksForTravel()`和`InitGame()`之间设置
- 对于多人游戏，服务器端的OptionsString会传递给客户端
- 如果需要修改，应该重写`InitGame()`等函数而不是直接修改这个变量

如果需要更复杂的参数处理，通常会创建自定义的GameMode子类并添加专门的解析逻辑。
## 总结
以上回答来自DeepSeek.
