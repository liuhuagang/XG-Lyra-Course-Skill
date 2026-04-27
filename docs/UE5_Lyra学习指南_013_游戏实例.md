# UE5_Lyra学习指南_013_游戏实例

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)

文章内容由小刚撰写,采用了以下多种方式:

1.口述转文字
2.AI重构
3.参考引擎源码
4.Lyra工程源码
5.结合社区论坛各位大佬的解析

- [UE5\_Lyra学习指南\_013\_游戏实例](#ue5_lyra学习指南_013_游戏实例)
	- [概述](#概述)
	- [LyraGameInstance](#lyragameinstance)
	- [UGameFrameworkComponentManager](#ugameframeworkcomponentmanager)
	- [引入GameplayTag](#引入gameplaytag)
	- [总结](#总结)



## 概述

本节主要玩了Lyra的GameInstance.这是一个非常重要的类!但是非必要,我们不要在这里写任何代码.
LyraGameInstance的父类UCommonGameInstance写了大量关于用于的消息管理.我们不在本节进行过多的关注.

``` txt
/**
* 游戏实例：运行中游戏实例的高级管理对象。
* 在游戏创建时生成，并在游戏实例关闭前不会被销毁。
* 若作为独立游戏运行，则会有一个此类对象。
* 若在编辑器中运行（即在 PIE 中运行），则每个 PIE 实例都会生成一个此类对象。
*/
```
它主要演示了一个关于客户端登录 向服务器请求验证的过程.
``` txt
	// 这是一个简单的示例，用于展示如何使用加密技术对游戏数据进行传输，并使用一个固定的密钥进行加密。
	// 若要实现完整的功能，您可能需要从安全的来源（例如通过 HTTPS 连接的网络服务）获取加密密钥。
	// 这种操作可以在本函数中完成，甚至可以异步进行——一旦获取到密钥，只需调用传入的响应委托即可。
```

## LyraGameInstance

在视频中,我们会启动两个单机实例,然后让其中一个主持会话,让另外一个去登录才能看到通信的过程.

``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "CommonGameInstance.h"

#include "LyraGameInstance.generated.h"

#define UE_API LYRAGAME_API

class ALyraPlayerController;
class UObject;

/**
 * Lyra项目的游戏实例
 */
UCLASS(MinimalAPI, Config = Game)
class ULyraGameInstance : public UCommonGameInstance
{
	GENERATED_BODY()

public:

	UE_API ULyraGameInstance(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	/**
	 * 获取此设备上的主玩家控制器（其他控制器为分屏子控制器）
	 * （必须具备有效的玩家状态）
	 * @参数 bRequiresValidUniqueId - 控制器是否也必须具有有效的唯一标识符（默认值为 true，以保持历史行为）
	 * @返回 此设备上的主控制器
	 * 
	 */
	// 因为这里的返回值不一样 所以不是override
	UE_API ALyraPlayerController* GetPrimaryPlayerController() const;

	// 是否可以加入会话
	UE_API virtual bool CanJoinRequestedSession() const override;

	// 处理用户得初始化
	UE_API virtual void HandlerUserInitialized(const UCommonUserInfo* UserInfo, bool bSuccess, FText Error, ECommonUserPrivilege RequestedPrivilege, ECommonUserOnlineContext OnlineContext) override;

	/** 处理加密密钥的设置事宜。若游戏需要自行实现此功能，则必须在自身（可能为异步）处理完成时调用该委托函数。*/
	UE_API virtual void ReceivedNetworkEncryptionToken(const FString& EncryptionToken, const FOnEncryptionKeyResponse& Delegate) override;

	/** 当客户端从服务器接收到“加密确认”控制消息时会调用此函数，通常会启用加密功能。*/
	UE_API virtual void ReceivedNetworkEncryptionAck(const FOnEncryptionKeyResponse& Delegate) override;

protected:
	/** virtual function to allow custom GameInstances an opportunity to set up what it needs */
	/** 一个虚拟函数，旨在为自定义的游戏实例提供设置自身所需内容的机会 */
	UE_API virtual void Init() override;

	/** 一个虚拟函数，旨在为自定义的游戏实例提供在关闭时进行清理操作的机会 */
	UE_API virtual void Shutdown() override;

	// 当客户端跳转到服务器的会话关卡时,我们可以在这里重写URL,携带令牌参数
	UE_API void OnPreClientTravelToSession(FString& URL);

	/** A hard-coded encryption key used to try out the encryption code. This is NOT SECURE, do not use this technique in production! */
	/** 一个硬编码的加密密钥，用于测试加密代码。此密钥并不安全，请勿在实际生产环境中使用此方法！*/
	TArray<uint8> DebugTestEncryptionKey;
};

#undef UE_API

```

UGameInstance::Init()
``` cpp

void UGameInstance::Init()
{
	TRACE_CPUPROFILER_EVENT_SCOPE(UGameInstance::Init);
	ReceiveInit();

	if (!IsRunningCommandlet())
	{
		UClass* SpawnClass = GetOnlineSessionClass();
		OnlineSession = NewObject<UOnlineSession>(this, SpawnClass);
		if (OnlineSession)
		{
			OnlineSession->RegisterOnlineDelegates();
		}

		if (!IsDedicatedServerInstance())
		{
			TSharedPtr<GenericApplication> App = FSlateApplication::Get().GetPlatformApplication();
			if (App.IsValid())
			{
				App->RegisterConsoleCommandListener(GenericApplication::FOnConsoleCommandListener::CreateUObject(this, &ThisClass::OnConsoleInput));
			}
		}

		FNetDelegates::OnReceivedNetworkEncryptionToken.BindUObject(this, &ThisClass::ReceivedNetworkEncryptionToken);
		FNetDelegates::OnReceivedNetworkEncryptionAck.BindUObject(this, &ThisClass::ReceivedNetworkEncryptionAck);
		FNetDelegates::OnReceivedNetworkEncryptionFailure.BindUObject(this, &ThisClass::ReceivedNetworkEncryptionFailure);

		IPlatformInputDeviceMapper& PlatformInputMapper = IPlatformInputDeviceMapper::Get();
		PlatformInputMapper.GetOnInputDeviceConnectionChange().AddUObject(this, &UGameInstance::HandleInputDeviceConnectionChange);
		PlatformInputMapper.GetOnInputDevicePairingChange().AddUObject(this, &UGameInstance::HandleInputDevicePairingChange);
	}

	SubsystemCollection.Initialize(this);
}
```


## UGameFrameworkComponentManager
``` txt
/** 
 * GameFrameworkComponentManager
 *
 * A manager to handle putting components on actors as they come and go.
 * Put in a request to instantiate components of a given class on actors of a given class and they will automatically be made for them as the actors are spawned.
 * Submit delegate handlers to listen for actors of a given class. Those handlers will automatically run when actors of a given class or registered as receivers or game events are sent.
 * Actors must opt-in to this behavior by calling AddReceiver/RemoveReceiver for themselves when they are ready to receive the components and when they want to remove them.
 * Any actors that are in memory when a request is made will automatically get the components, and any in memory when a request is removed will lose the components immediately.
 * Requests are reference counted, so if multiple requests are made for the same actor class and component class, only one component will be added and that component wont be removed until all requests are removed.
 */
/**
* 游戏框架组件管理器*
* 一名管理人员负责在角色出现和消失时为其放置组件。
* 提出请求以在具有特定类别的角色上实例化特定类别的组件，当角色被生成时，这些组件会自动为其创建。
* 提交委托处理程序以监听具有特定类别的角色。当具有特定类别的角色（或注册为接收者或游戏事件的此类角色）被发送时，这些处理程序将自动运行。
* 角色必须通过调用“添加接收者/移除接收者”方法来选择参与此行为，当它们准备好接收组件以及想要移除组件时。
* 当请求被发出时，处于内存中的任何角色都会自动获得组件，而当请求被撤销时，处于内存中的任何角色都会立即失去组件。
* 请求是引用计数的，因此如果为同一角色类和组件类发出多个请求，则只会添加一个组件，并且该组件在所有请求都被撤销之前不会被移除。
*/
```
这个GameFrameworkComponentManager也非常重要.
我们后面可以看到关于角色初始化推进过程会大量使用到它的函数.

## 引入GameplayTag
有很多方式去导入GameplayTag.
这里主要演示通过C++导入.
还有ini导入,手动添加,DataTable等方式.
此处写完LyraGameplayTags.h文件即可.
使用特性上一套课程已经完全讲了.不再赘述.官方文档也讲的很详细了.
## 总结
本节主要完善了GameInstance,并展示了其中客户端与服务加密通信的案例.
注意该功能需要通过命令行开启调试参数,并使用两个单独的实例进行交互才可以看到效果.

