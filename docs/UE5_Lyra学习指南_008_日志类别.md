# UE5_Lyra学习指南_008_日志类别

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
- [UE5\_Lyra学习指南\_008\_日志类别](#ue5_lyra学习指南_008_日志类别)
	- [观前提示](#观前提示)
	- [1.创建日志定义](#1创建日志定义)
	- [2.复习部分知识点](#2复习部分知识点)
	- [3.讲解获取上下文对象方法](#3讲解获取上下文对象方法)
	- [总结](#总结)
	- [参考文献](#参考文献)

本节核心步骤:
1.创建日志文件
2.复习部分知识点
3.讲解LyraLogChannels.h中的世界上下文方法

## 观前提示
这里正式章节的第一课.
我们有一个专门的交流群[Lyra深度交流学习群](1051436231),需要验证订单号方可入群.
订单号,在B站购买课程后,B站课堂的官方号会私信发送订单号.不是渠道支付的订单号.不是渠道支付的商户单号.
以下是参考格式:
20250629*********.
制作课程不易,还请大家支持正版~~

## 1.创建日志定义
在.h里面写
``` cpp
LYRAGAME_API DECLARE_LOG_CATEGORY_EXTERN(LogLyra, Log, All);
```
在.cpp里面写
``` cpp
DEFINE_LOG_CATEGORY(LogLyra);
```
这样定义的日志可以在其他文件进行使用.
需要修改的文件有:
LyraEditor.h
```
#pragma once

#include "Logging/LogMacros.h"

DECLARE_LOG_CATEGORY_EXTERN(LogLyraEditor, Log, All);
```
LyraGamePhaseLog.h
```
DECLARE_LOG_CATEGORY_EXTERN(LogLyraGamePhase, Log, All);
```
LyraLogChannels.h
```
LYRAGAME_API DECLARE_LOG_CATEGORY_EXTERN(LogLyra, Log, All);
LYRAGAME_API DECLARE_LOG_CATEGORY_EXTERN(LogLyraExperience, Log, All);
LYRAGAME_API DECLARE_LOG_CATEGORY_EXTERN(LogLyraAbilitySystem, Log, All);
LYRAGAME_API DECLARE_LOG_CATEGORY_EXTERN(LogLyraTeams, Log, All);
```

LyraCheatManager.h
```
DECLARE_LOG_CATEGORY_EXTERN(LogLyraCheat, Log, All);
```

LyraGameSettingRegistry.h
```
DECLARE_LOG_CATEGORY_EXTERN(LogLyraGameSettingRegistry, Log, Log);
```

LyraReplicationGraph.h
```
DECLARE_LOG_CATEGORY_EXTERN(LogLyraRepGraph, Display, All);
```

## 2.复习部分知识点

该日志文件位于引擎源码位置:
E:\Epic\UE\UE_5.6\Engine\Source\Runtime\Core\Public\Logging\LogMacros.h

1.带有static的区分
注意区分
``` cpp
DEFINE_LOG_CATEGORY_STATIC(SourceFilterPresets, Display, Display);
```
带有Static的日志定义宏
DEFINE_LOG_CATEGORY_STATIC:
一个宏，用于将日志类别定义为 C++ 中的“静态”变量。此宏应仅在源文件中声明。仅能在该单个文件中访问。

DEFINE_LOG_CATEGORY_STATIC

2.多线程不安全

你可以继承 FOutputDevice，实现你自己的输出类。

3.日志详细级别
日志详细级别	打印在控制台中	打印在编辑器日志中	文本颜色	其他信息
致命（Fatal）	是	不适用	不适用	会话崩溃。
错误（Error）	是	是	红色	不适用
警告（Warning）	是	是	黄色	不适用
显示（Display）	是	是	灰色	不适用
日志（Log）	否	是	灰色	不适用
冗长（Verbose）	否	否	不适用	不适用
极其冗长（VeryVerbose）	否	否	不适用	可使用日志掩码和特殊枚举值设置文本颜色。

注意Fatal会导致崩溃
对应的日志定义在
E:\Epic\UE\UE_5.6\Engine\Source\Runtime\Core\Public\Logging\LogVerbosity.h

## 3.讲解获取上下文对象方法
``` cpp
FString GetClientServerContextString(UObject* ContextObject = nullptr);
```
该函数在LyraExperienceManagerComponent.h中调用,用来打印Experience的加载日志.传递的对象是GameState.该对象具有网络同步的属性.

比较有意思的是这里面有一个全局变量GPlayInEditorContextString
``` cpp
#if WITH_EDITOR
// A debugging aid set when we switch out different play worlds during Play In Editor / PIE
// 在“编辑器中游戏”（PIE）模式下切换不同的游戏场景时会启用此调试辅助工具集
ENGINE_API FString GPlayInEditorContextString(TEXT("invalid"));
#endif
```
它主要是在编辑器下通过通过上下文对象获取调试字段.
``` cpp
// Update the debugging aid GPlayInEditorContextString based on the current world context (does nothing in WITH_EDITOR=0 builds)
// 根据当前的世界环境更新调试辅助工具 GPlayInEditorContextString（在 WITH_EDITOR=0 构建模式下不执行任何操作）
ENGINE_API void UpdatePlayInEditorWorldDebugString(const FWorldContext* WorldContext);

// Returns the Debug string for a given world (Standalone, Listen Server, Client #, etc)
// 返回指定世界的调试字符串（如独立运行模式、监听服务器模式、客户端编号等）
ENGINE_API FString GetDebugStringForWorld(const UWorld* World);
```
UpdatePlayInEditorWorldDebugString()这个函数的主要调用时间都是在场景世界切换,加载地图时调用.

回到Lyra项目中,GPlayInEditorContextString在UGameplayMessageSubsystem::BroadcastMessageInternal也有访问.主要为了调试时,获取消息发生的世界类型.

此处补充一段关于FWorldContext的注释,来自引擎源码:
``` txt
	/*
	* FWorldContext 是用于在引擎层面处理 UWorld 的一个上下文环境。当引擎启动和销毁世界时，我们需要一种方法来明确区分哪些世界属于哪个世界。
	* WorldContext 可以被视为一条轨道。默认情况下，我们有一个轨道用于加载和卸载关卡。添加第二个上下文就是添加第二条轨道；为世界提供另一个存在的进展轨道。
	* 对于游戏引擎，直到我们决定支持多个同时存在的世界之前，将只有一个 WorldContext。对于编辑器引擎，可能会有一个用于编辑器世界和一个用于 PIE 世界的 WorldContext。
	* FWorldContext 提供了管理“当前的 PIE UWorld*”的方法，以及与连接/前往新世界相关的状态。
	* FWorldContext 应该保持在 UEngine 类内部。外部代码不应保留指针或直接管理 FWorldContext。外部代码仍然可以处理 UWorld*，并将 UWorld* 传递给引擎级别的函数。引擎代码可以根据给定的 UWorld* 查找相关的上下文。
	* 为了方便起见，FWorldContext 可以维护外部对 UWorld* 的指针。例如，PIE 可以将 UWorld* UEditorEngine:：PlayWorld 与 PIE 世界上下文关联起来。如果 PIE UWorld 发生变化，UEditorEngine::PlayWorld 指针将自动更新。这是通过调用 AddRef() 和 SetCurrentWorld() 来实现的。
	*/

```

## 总结


## 参考文献

[虚幻中的日志记录](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/logging-in-unreal-engine)

