# UE5_Lyra学习指南_019_玩家生成管理组件

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_019\_玩家生成管理组件](#ue5_lyra学习指南_019_玩家生成管理组件)
	- [概述](#概述)
	- [LyraPlayerSpawningManagerComponent](#lyraplayerspawningmanagercomponent)
		- [检查父类](#检查父类)
		- [PIE就近生成出生点](#pie就近生成出生点)
		- [出生点算法](#出生点算法)
	- [代码](#代码)
	- [总结](#总结)



## 概述
LyraPlayerSpawningManagerComponent接受GameMode的转发,解耦选择玩家出生点的这个功能.

## LyraPlayerSpawningManagerComponent
### 检查父类
通过InitializeComponent()方法.  
1.绑定在关卡添加的代理进行检查新加载进入的出生点  
2.绑定在世界新生成Actor的代理处理动态生成的出生点
``` cpp
void ULyraPlayerSpawningManagerComponent::InitializeComponent()
{
	Super::InitializeComponent();

	// 当有个一个关卡被添加到世界中了,我们需要处理里面的玩家出生点
	FWorldDelegates::LevelAddedToWorld.AddUObject(this, &ThisClass::OnLevelAdded);

	UWorld* World = GetWorld();
	// 绑定好世界新增的对象
	World->AddOnActorSpawnedHandler(FOnActorSpawned::FDelegate::CreateUObject(this, &ThisClass::HandleOnActorSpawned));

	// 捕获场景中的玩家出生点
	for (TActorIterator<ALyraPlayerStart> It(World); It; ++It)
	{
		if (ALyraPlayerStart* PlayerStart = *It)
		{
			CachedPlayerStarts.Add(PlayerStart);
		}
	}
}
```

### PIE就近生成出生点
在GameMode生成玩家出生点时.如果是PIE模式下.回去找有没有PIE模式下的出生点.

```cpp

AActor* ULyraPlayerSpawningManagerComponent::ChoosePlayerStart(AController* Player)
{
	if (Player)
	{
#if WITH_EDITOR
		// 这里寻找的是编辑器的专属最近生成点APlayerStartPIE
		if (APlayerStart* PlayerStart = FindPlayFromHereStart(Player))
		{
			return PlayerStart;
		}
#endif
	//......
	}
}

```

``` cpp
#if WITH_EDITOR
APlayerStart* ULyraPlayerSpawningManagerComponent::FindPlayFromHereStart(AController* Player)
{
	// Only 'Play From Here' for a player controller, bots etc. should all spawn from normal spawn points.
	// 只有玩家控制器、机器人等才应从常规的起始点生成，而“从这里开始游戏”选项则仅适用于玩家控制器。
	if (Player->IsA<APlayerController>())
	{
		if (UWorld* World = GetWorld())
		{
			for (TActorIterator<APlayerStart> It(World); It; ++It)
			{
				if (APlayerStart* PlayerStart = *It)
				{
					if (PlayerStart->IsA<APlayerStartPIE>())
					{
						// Always prefer the first "Play from Here" PlayerStart, if we find one while in PIE mode
						// 当处于 PIE 模式时，如果能找到“从这里开始播放”的首个 PlayerStart 功能，则始终优先使用它。
						return PlayerStart;
					}
				}
			}
		}
	}

	return nullptr;
}
#endif
```

``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.

//=============================================================================
// Player start for PIE - can be spawned during play.
//=============================================================================

#pragma once

#include "CoreMinimal.h"
#include "UObject/ObjectMacros.h"
#include "GameFramework/PlayerStart.h"
#include "PlayerStartPIE.generated.h"

UCLASS(notplaceable,MinimalAPI)
class APlayerStartPIE : public APlayerStart
{
	GENERATED_UCLASS_BODY()

};

```

### 出生点算法
不考虑被完全阻塞的出生点.从可用的出生点随机选一个即可.
``` cpp
	// Utility
	// 找到第一个随机的未被占用的玩家出生点
	UE_API APlayerStart* GetFirstRandomUnoccupiedPlayerStart(AController* Controller, const TArray<ALyraPlayerStart*>& FoundStartPoints) const;
```

这里保留了很多的拓展空间.比如蓝图覆写的事件等等.
## 代码
ULyraPlayerSpawningManagerComponent:  
``` cpp

/**
 * @class ULyraPlayerSpawningManagerComponent
 *
 * 用来辅助GameMode管理玩家的生成
 * 
 */
UCLASS(MinimalAPI)
class ULyraPlayerSpawningManagerComponent : public UGameStateComponent
{
	GENERATED_BODY()

public:
	UE_API ULyraPlayerSpawningManagerComponent(const FObjectInitializer& ObjectInitializer);

	/** UActorComponent */
	UE_API virtual void InitializeComponent() override;
	UE_API virtual void TickComponent(float DeltaTime, enum ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;
	/** ~UActorComponent */

protected:
	// Utility
	// 找到第一个随机的未被占用的玩家出生点
	UE_API APlayerStart* GetFirstRandomUnoccupiedPlayerStart(AController* Controller, const TArray<ALyraPlayerStart*>& FoundStartPoints) const;

	// 自定义的筛选规则,需要自行拓展
	virtual AActor* OnChoosePlayerStart(AController* Player, TArray<ALyraPlayerStart*>& PlayerStarts) { return nullptr; }
	// 自定义的玩家完成重启事件
	virtual void OnFinishRestartPlayer(AController* Player, const FRotator& StartRotation) { }

	// 用于蓝图拓展的自定义玩家完成重启事件
	UFUNCTION(BlueprintImplementableEvent, meta=(DisplayName=OnFinishRestartPlayer))
	UE_API void K2_OnFinishRestartPlayer(AController* Player, const FRotator& StartRotation);

private:

	/** We proxy these calls from ALyraGameMode, to this component so that each experience can more easily customize the respawn system they want. */
	/** 我们从 AlyraGameMode 中对这些调用进行代理处理，使其传递至本组件，这样每个体验就能更方便地根据自身需求定制重生系统。*/
	UE_API AActor* ChoosePlayerStart(AController* Player);
	UE_API bool ControllerCanRestart(AController* Player);
	UE_API void FinishRestartPlayer(AController* NewPlayer, const FRotator& StartRotation);
	friend class ALyraGameMode;
	/** ~ALyraGameMode */

	UPROPERTY(Transient)
	TArray<TWeakObjectPtr<ALyraPlayerStart>> CachedPlayerStarts;

private:
	// 处理新增关卡的玩家出生点逻辑
	UE_API void OnLevelAdded(ULevel* InLevel, UWorld* InWorld);
	// 处理动态生成的玩家出生点
	UE_API void HandleOnActorSpawned(AActor* SpawnedActor);

#if WITH_EDITOR
	// PIE模型下寻找最近出生点
	UE_API APlayerStart* FindPlayFromHereStart(AController* Player);
#endif
};

```

## 总结
这个类属于是留给我们自己拓展的类.可以看到在选择出生点时,以及玩家完成重生都留了函数接口,以供我们嵌入其他逻辑.
