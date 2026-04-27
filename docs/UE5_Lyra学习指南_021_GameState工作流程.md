# UE5_Lyra学习指南_021_GameState工作流程

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_021\_GameState工作流程](#ue5_lyra学习指南_021_gamestate工作流程)
	- [概述](#概述)
	- [LyraGameState的功能](#lyragamestate的功能)
		- [创建组件](#创建组件)
		- [获取服务端帧率](#获取服务端帧率)
		- [修改回放玩家状态类](#修改回放玩家状态类)
		- [同步服务端客户端信息](#同步服务端客户端信息)
	- [代码](#代码)
		- [LyraGameState](#lyragamestate)
		- [LyraVerbMessage](#lyraverbmessage)
		- [LyraVerbMessageHelpers](#lyraverbmessagehelpers)
		- [LyraAbilitySystemComponent](#lyraabilitysystemcomponent)
	- [总结](#总结)



## 概述
本节主要讲述LyraGameState的工作代码.  
我们会一口气把这个类搞定.但是这个类里面有一部分代码设计到了GAS和回放.同Experience的讲解思路一样.我们也会被这些方法写好.
但是不在这节对他们的全流程进行阐述.

## LyraGameState的功能
### 创建组件
创建GAS组件
创建Experience组件
```cpp
ALyraGameState::ALyraGameState(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
{
	PrimaryActorTick.bCanEverTick = true;
	PrimaryActorTick.bStartWithTickEnabled = true;

	AbilitySystemComponent = ObjectInitializer.CreateDefaultSubobject<ULyraAbilitySystemComponent>(this, TEXT("AbilitySystemComponent"));
	AbilitySystemComponent->SetIsReplicated(true);
	AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);

	ExperienceManagerComponent = CreateDefaultSubobject<ULyraExperienceManagerComponent>(TEXT("ExperienceManagerComponent"));

	ServerFPS = 0.0f;
}
```
此时简单复习下GameplayEffectReplicationMode,更多内容推荐插件GitHub的GAS中文文档!
```cpp
/** How gameplay effects will be replicated to clients */
/** 游戏玩法的执行效果将如何传递给客户端 */
UENUM()
enum class EGameplayEffectReplicationMode : uint8
{
	/** Only replicate minimal gameplay effect info. Note: this does not work for Owned AbilitySystemComponents (Use Mixed instead). */
	/** 仅复制最基础的游戏玩法效果信息。注意：此方法不适用于已拥有的能力系统组件（请使用混合方法代替）。*/
	Minimal,
	/** Only replicate minimal gameplay effect info to simulated proxies but full info to owners and autonomous proxies */
	/** 仅将游戏玩法的最基本信息复制到模拟代理中，而将完整信息复制给所有者和自主代理 */
	Mixed,
	/** Replicate full gameplay info to all */
	/** 将完整的游戏信息复制给所有玩家 */
	Full,
};
```
记得初始化ASC的ActorInfo:
``` cpp
void ALyraGameState::PostInitializeComponents()
{
	Super::PostInitializeComponents();

	check(AbilitySystemComponent);

	// 初始化ASC的组件信息
	AbilitySystemComponent->InitAbilityActorInfo(/*Owner=*/ this, /*Avatar=*/ this);
}
```

### 获取服务端帧率
在UnrealEngine.cpp中有几个专门暴露出来的变量供我们使用

``` cpp
// We expose these variables to everyone as we need to access them in other files via an extern
// 我们将这些变量公开给所有人，因为我们需要在其他文件中通过“extern”关键字来访问它们。
ENGINE_API float GAverageFPS = 0.0f;
ENGINE_API float GAverageMS = 0.0f;
ENGINE_API double GLastMemoryWarningTime = 0.f;
ENGINE_API bool GIsLowMemory = false;

```

通过属性同步,在Tick里面从服务端修改,然后传递给客户端
``` cpp


void ALyraGameState::Tick(float DeltaSeconds)
{
	Super::Tick(DeltaSeconds);

	if (GetLocalRole() == ROLE_Authority)
	{
		ServerFPS = GAverageFPS;
	}
}

```
``` cpp
public:
	// Gets the server's FPS, replicated to clients
	// 获取服务器的帧率，这是由服务器同步过来的
	UE_API float GetServerFPS() const;
protected:
	UPROPERTY(Replicated)
	float ServerFPS;

```

### 修改回放玩家状态类
``` cpp
public:
	// Indicate the local player state is recording a replay
	// 表示本地玩家状态正在录制回放
	UE_API void SetRecorderPlayerState(APlayerState* NewPlayerState);

	// Gets the player state that recorded the replay, if valid
	// 获取记录回放时所使用的玩家状态（如果该状态有效）
	UE_API APlayerState* GetRecorderPlayerState() const;

	// Delegate called when the replay player state changes
	// 当回放播放器状态发生变化时触发的回调函数 在玩家控制器里处理
	DECLARE_MULTICAST_DELEGATE_OneParam(FOnRecorderPlayerStateChanged, APlayerState*);
	FOnRecorderPlayerStateChanged OnRecorderPlayerStateChangedEvent;
protected:
	// The player state that recorded a replay, it is used to select the right pawn to follow
	// This is only set in replay streams and is not replicated normally
	// 玩家所记录的回放状态，用于选择正确的棋子进行跟随
	// 这种状态仅在回放流中设置，通常不会进行复制操作
	UPROPERTY(Transient, ReplicatedUsing = OnRep_RecorderPlayerState)
	TObjectPtr<APlayerState> RecorderPlayerState;

	UFUNCTION()
	UE_API void OnRep_RecorderPlayerState();	

```

记得以上网络同步变量都需要标记生命周期
``` cpp


void ALyraGameState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	DOREPLIFETIME(ThisClass, ServerFPS);
	// 此属性仅用于发送至回放连接
	DOREPLIFETIME_CONDITION(ThisClass, RecorderPlayerState, COND_ReplayOnly);
}
```


### 同步服务端客户端信息
``` cpp
	// Send a message that all clients will (probably) get
	// (use only for client notifications like eliminations, server join messages, etc... that can handle being lost)
	// 发送一条消息，告知所有客户端（很可能）都会接收到该消息
	// （仅用于客户端通知，例如淘汰通知、服务器加入消息等，这类消息可以丢失）
	// 服务器调用,所有客户端执行
	UFUNCTION(NetMulticast, Unreliable, BlueprintCallable, Category = "Lyra|GameState")
	UE_API void MulticastMessageToClients(const FLyraVerbMessage Message);

	// Send a message that all clients will be guaranteed to get
	// (use only for client notifications that cannot handle being lost)
	// 发送一条消息，确保所有客户端都能接收到该消息
	// （仅用于那些无法承受消息丢失情况的客户端通知）
	// 服务器调用 所有客户端执行
	UFUNCTION(NetMulticast, Reliable, BlueprintCallable, Category = "Lyra|GameState")
	UE_API void MulticastReliableMessageToClients(const FLyraVerbMessage Message);

```


## 代码

### LyraGameState

``` cpp

/**
 * ALyraGameState
 *
 *	The base game state class used by this project.
 *	该项目所使用的基础游戏状态类。
 */
UCLASS(MinimalAPI, Config = Game)
class ALyraGameState : public AModularGameStateBase, public IAbilitySystemInterface
{
	GENERATED_BODY()

public:
	/*
	 * 构造函数
	 * 1.开启Tick的功能
	 * 2.初始化GAS组件 设置GAS的同步模式
	 * 3.创建体验管理组件 负载加载Experience
	 */
	UE_API ALyraGameState(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	//~AActor interface
	// 暂无功能
	UE_API virtual void PreInitializeComponents() override;
	// 暂无功能
	UE_API virtual void PostInitializeComponents() override;
	// 暂无功能
	UE_API virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
	// 从服务端读取帧数通过属性同步到客户端
	UE_API virtual void Tick(float DeltaSeconds) override;
	//~End of AActor interface

	//~AGameStateBase interface
	
	/** 将玩家状态添加到玩家数组中  暂无功能*/
	UE_API virtual void AddPlayerState(APlayerState* PlayerState) override;
	
	/** 从玩家数组中移除玩家状态信息。暂无功能 */
	UE_API virtual void RemovePlayerState(APlayerState* PlayerState) override;
	
	/**
	 * 在无缝旅行过渡过程中被调用两次（一次是在过渡地图加载时，一次是在目的地地图加载时）
	 * 主要用于移除无效的用户和机器人
	 * 
	 */
	UE_API virtual void SeamlessTravelTransitionCheckpoint(bool bToTransitionMap) override;
	
	//~End of AGameStateBase interface

	//~IAbilitySystemInterface
	// 获取ASC组件
	UE_API virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;
	//~End of IAbilitySystemInterface

	// Gets the ability system component used for game wide things
	// 获取用于全局游戏操作的能力系统组件
	UFUNCTION(BlueprintCallable, Category = "Lyra|GameState")
	ULyraAbilitySystemComponent* GetLyraAbilitySystemComponent() const { return AbilitySystemComponent; }

	// Send a message that all clients will (probably) get
	// (use only for client notifications like eliminations, server join messages, etc... that can handle being lost)
	// 发送一条消息，告知所有客户端（很可能）都会接收到该消息
	// （仅用于客户端通知，例如淘汰通知、服务器加入消息等，这类消息可以丢失）
	// 服务器调用,所有客户端执行
	UFUNCTION(NetMulticast, Unreliable, BlueprintCallable, Category = "Lyra|GameState")
	UE_API void MulticastMessageToClients(const FLyraVerbMessage Message);

	// Send a message that all clients will be guaranteed to get
	// (use only for client notifications that cannot handle being lost)
	// 发送一条消息，确保所有客户端都能接收到该消息
	// （仅用于那些无法承受消息丢失情况的客户端通知）
	// 服务器调用 所有客户端执行
	UFUNCTION(NetMulticast, Reliable, BlueprintCallable, Category = "Lyra|GameState")
	UE_API void MulticastReliableMessageToClients(const FLyraVerbMessage Message);

	// Gets the server's FPS, replicated to clients
	// 获取服务器的帧率，这是由服务器同步过来的
	UE_API float GetServerFPS() const;

	// Indicate the local player state is recording a replay
	// 表示本地玩家状态正在录制回放
	UE_API void SetRecorderPlayerState(APlayerState* NewPlayerState);

	// Gets the player state that recorded the replay, if valid
	// 获取记录回放时所使用的玩家状态（如果该状态有效）
	UE_API APlayerState* GetRecorderPlayerState() const;

	// Delegate called when the replay player state changes
	// 当回放播放器状态发生变化时触发的回调函数 在玩家控制器里处理
	DECLARE_MULTICAST_DELEGATE_OneParam(FOnRecorderPlayerStateChanged, APlayerState*);
	FOnRecorderPlayerStateChanged OnRecorderPlayerStateChangedEvent;

private:
	// Handles loading and managing the current gameplay experience
	// 负责加载并管理当前的游戏体验
	UPROPERTY()
	TObjectPtr<ULyraExperienceManagerComponent> ExperienceManagerComponent;

	// The ability system component subobject for game-wide things (primarily gameplay cues)
	// 用于游戏范围内各类事物（主要是游戏提示）的能力系统子对象
	UPROPERTY(VisibleAnywhere, Category = "Lyra|GameState")
	TObjectPtr<ULyraAbilitySystemComponent> AbilitySystemComponent;

protected:
	UPROPERTY(Replicated)
	float ServerFPS;

	// The player state that recorded a replay, it is used to select the right pawn to follow
	// This is only set in replay streams and is not replicated normally
	// 玩家所记录的回放状态，用于选择正确的棋子进行跟随
	// 这种状态仅在回放流中设置，通常不会进行复制操作
	UPROPERTY(Transient, ReplicatedUsing = OnRep_RecorderPlayerState)
	TObjectPtr<APlayerState> RecorderPlayerState;

	// 用于属性网络同步过来之后进行处理.
	UFUNCTION()
	UE_API void OnRep_RecorderPlayerState();

};

```



### LyraVerbMessage
``` cpp

// Represents a generic message of the form Instigator Verb Target (in Context, with Magnitude)
// 表示一种通用的消息形式，即“起因 + 动词 + 目标（在特定情境下，包含强度）* 本项目所使用的基础能力系统组件类”
USTRUCT(BlueprintType)
struct FLyraVerbMessage
{
	GENERATED_BODY()

	// 动词 表示行为的Tag
	UPROPERTY(BlueprintReadWrite, Category=Gameplay)
	FGameplayTag Verb;

	// 主语 发起者的指针
	UPROPERTY(BlueprintReadWrite, Category=Gameplay)
	TObjectPtr<UObject> Instigator = nullptr;

	// 宾语 接受者的指针
	UPROPERTY(BlueprintReadWrite, Category=Gameplay)
	TObjectPtr<UObject> Target = nullptr;

	// 定语 发起者的状态Tag
	UPROPERTY(BlueprintReadWrite, Category=Gameplay)
	FGameplayTagContainer InstigatorTags;

	// 定语 接受者的状态Tag
	UPROPERTY(BlueprintReadWrite, Category=Gameplay)
	FGameplayTagContainer TargetTags;

	// 壮语 上下文的Tag
	UPROPERTY(BlueprintReadWrite, Category=Gameplay)
	FGameplayTagContainer ContextTags;

	// 量级 表示具体的数量
	UPROPERTY(BlueprintReadWrite, Category=Gameplay)
	double Magnitude = 1.0;

	// Returns a debug string representation of this message
	// 返回此消息的调试字符串表示形式
	LYRAGAME_API FString ToString() const;
};


```

### LyraVerbMessageHelpers
``` cpp
/*
 * 蓝图静态函数库
 * 主要用于通讯Message的处理
 */
UCLASS(MinimalAPI)
class ULyraVerbMessageHelpers : public UBlueprintFunctionLibrary
{
	GENERATED_BODY()

public:
	// 获取玩家状态
	UFUNCTION(BlueprintCallable, Category = "Lyra")
	static UE_API APlayerState* GetPlayerStateFromObject(UObject* Object);

	// 获取玩家控制器
	UFUNCTION(BlueprintCallable, Category = "Lyra")
	static UE_API APlayerController* GetPlayerControllerFromObject(UObject* Object);

	// 将Message转换城Cue参数
	UFUNCTION(BlueprintCallable, Category = "Lyra")
	static UE_API FGameplayCueParameters VerbMessageToCueParameters(const FLyraVerbMessage& Message);

	// 将Cue参数转换城Message
	UFUNCTION(BlueprintCallable, Category = "Lyra")
	static UE_API FLyraVerbMessage CueParametersToVerbMessage(const FGameplayCueParameters& Params);
};



```
### LyraAbilitySystemComponent
``` cpp
/**
 * ULyraAbilitySystemComponent
 *
 *	Base ability system component class used by this project.
 * 本项目所使用的基础能力系统组件类。
 */
UCLASS(MinimalAPI)
class ULyraAbilitySystemComponent : public UAbilitySystemComponent
{
	GENERATED_BODY()

public:

	UE_API ULyraAbilitySystemComponent(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

};



```

## 总结
在有了GameMode的基础之后,来处理GameState就相对容易许多了.
如果涉及更多引擎流程,大家也可以自行去引擎函数中打断点,查看执行时机与功能.