# UE5_Lyra学习指南_023_异步询问体验加载

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_023\_异步询问体验加载](#ue5_lyra学习指南_023_异步询问体验加载)
	- [概述](#概述)
	- [AsyncAction\_ExperienceReady](#asyncaction_experienceready)
		- [GameStateEvent](#gamestateevent)
		- [延时下一帧处理](#延时下一帧处理)
		- [销毁](#销毁)
	- [代码](#代码)
	- [总结](#总结)



## 概述
这是一个简单的异步蓝图节点的写法.在上套课程中我们已经写了很多次了,比如TTS的异步合成多线程多用.
## AsyncAction_ExperienceReady
代码比较简单,我们可以通过蓝图打印字符串测试这个节点.
有几个细节需要注意.
### GameStateEvent
就是当我们通过世界还找不到GameState的时候,可以通过GameStateEvent这个事件等待GameState的生成回调!
``` cpp
void UAsyncAction_ExperienceReady::Activate()
{
	if (UWorld* World = WorldPtr.Get())
	{
		if (AGameStateBase* GameState = World->GetGameState())
		{
			// 有GameState 直接通过GameState实现
			Step2_ListenToExperienceLoading(GameState);
		}
		else
		{
			// GameState还没生成,绑定在GameState生成事件上
			World->GameStateSetEvent.AddUObject(this, &ThisClass::Step1_HandleGameStateSet);
		}
	}
	else
	{
		// No world so we'll never finish naturally
		// 没有世界,无法执行,直接销毁
		SetReadyToDestroy();
	}
}

```
### 延时下一帧处理
当我们考虑蓝图执行询问时,需要区分这一帧和下一帧.因为不区分的话,会修改蓝图的主线程逻辑!
``` cpp
void UAsyncAction_ExperienceReady::Step2_ListenToExperienceLoading(AGameStateBase* GameState)
{
	check(GameState);
	ULyraExperienceManagerComponent* ExperienceComponent = GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
	check(ExperienceComponent);
	// 是否已经加载了 如果加载了就直接调用 但是直接调用可能会有主逻辑问题 那么最好手动做一帧的延迟
	// 没有加载就绑定在代理上即可
	if (ExperienceComponent->IsExperienceLoaded())
	{
		UWorld* World = GameState->GetWorld();
		check(World);

		// The experience happened to be already loaded, but still delay a frame to
		// make sure people don't write stuff that relies on this always being true

		// 这段代码中的逻辑已经得到了执行，但为了确保不会出现因某些操作依赖于该逻辑始终为真而导致的问题，还是再延迟了一帧。

		
		//@TODO: Consider not delaying for dynamically spawned stuff / any time after the loading screen has dropped?
		//@TODO: Maybe just inject a random 0-1s delay in the experience load itself?

		//@待办事项：考虑不要为动态生成的内容或在加载屏幕消失之后的任何时间点进行延迟处理？
		//@待办事项：或许可以在游戏加载过程中直接插入一个 0 到 1 秒的随机延迟？

		
		World->GetTimerManager().SetTimerForNextTick(FTimerDelegate::CreateUObject(this, &ThisClass::Step4_BroadcastReady));
	}
	else
	{
		ExperienceComponent->CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::Step3_HandleExperienceLoaded));
	}
}
```
### 销毁
一定要记得对象的生命周期销毁!
我们是在创建时绑定在World上面的,之前我们写异步节点时绑定在GameInstance上面的.
``` cpp
UAsyncAction_ExperienceReady* UAsyncAction_ExperienceReady::WaitForExperienceReady(UObject* InWorldContextObject)
{
	UAsyncAction_ExperienceReady* Action = nullptr;

	if (UWorld* World = GEngine->GetWorldFromContextObject(InWorldContextObject, EGetWorldErrorMode::LogAndReturnNull))
	{
		Action = NewObject<UAsyncAction_ExperienceReady>();
		Action->WorldPtr = World;
		// 这里是注册到了世界上
		Action->RegisterWithGameInstance(World);
	}

	return Action;
}
```
销毁时机.
一般我们会有一个统一的销毁入口,但是如由于逻辑控制没有走到这里,也请提前进行调用销毁,避免内存溢出.
``` cpp
void UAsyncAction_ExperienceReady::Step4_BroadcastReady()
{
	// 触发代理!
	OnReady.Broadcast();

	SetReadyToDestroy();
}

```
``` cpp
void UAsyncAction_ExperienceReady::Activate()
{
	if (UWorld* World = WorldPtr.Get())
	{
		// ....
	}
	else
	{
		// No world so we'll never finish naturally
		// 没有世界,无法执行,直接销毁
		SetReadyToDestroy();
	}
}

```

## 代码
``` cpp

DECLARE_DYNAMIC_MULTICAST_DELEGATE(FExperienceReadyAsyncDelegate);

/**
 * Asynchronously waits for the game state to be ready and valid and then calls the OnReady event.  Will call OnReady
 * immediately if the game state is valid already.
 *
 * 异步等待游戏状态准备好且有效，然后调用“就绪”事件。  如果游戏状态已经有效，则会立即调用“就绪”事件。
 */
UCLASS()
class UAsyncAction_ExperienceReady : public UBlueprintAsyncActionBase
{
	GENERATED_UCLASS_BODY()

public:
	// Waits for the experience to be determined and loaded
	// 等待体验结果被确定并加载完成 蓝图函数乳沟
	UFUNCTION(BlueprintCallable, meta=(WorldContext = "WorldContextObject", BlueprintInternalUseOnly="true"))
	static UAsyncAction_ExperienceReady* WaitForExperienceReady(UObject* WorldContextObject);

	// 自动激活,由蓝图调用.如果是C++,请自行调用
	virtual void Activate() override;

public:

	// Called when the experience has been determined and is ready/loaded
	// 当体验内容已确定并准备就绪/加载完成时触发此事件
	UPROPERTY(BlueprintAssignable)
	FExperienceReadyAsyncDelegate OnReady;

private:
	// 步骤1 设置GameState
	void Step1_HandleGameStateSet(AGameStateBase* GameState);

	// 步骤2 绑定事件
	void Step2_ListenToExperienceLoading(AGameStateBase* GameState);
	
	// 步骤3 处理Experience加载完毕
	void Step3_HandleExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience);

	// 步骤四 广播并销毁自身
	void Step4_BroadcastReady();

	TWeakObjectPtr<UWorld> WorldPtr;
};


```


## 总结
首先一定要记得生命周期,然后去销毁它.
其次一定记得错帧处理!因为它是个异步节点.不错过一帧,会导致主线程的执行业务逻辑的顺序混乱!而且很难排查!