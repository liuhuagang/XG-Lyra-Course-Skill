# UE5_Lyra学习指南_024_PlayerState工作流程

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_024\_PlayerState工作流程](#ue5_lyra学习指南_024_playerstate工作流程)
	- [概述](#概述)
	- [LyraPlayerState](#lyraplayerstate)
		- [区分玩家连接是否活跃](#区分玩家连接是否活跃)
		- [设置PawnData](#设置pawndata)
		- [设置队伍](#设置队伍)
			- [设置队伍](#设置队伍-1)
			- [下级传递](#下级传递)
			- [队伍与战队ID](#队伍与战队id)
		- [观战角度的同步](#观战角度的同步)
		- [服务器与客户端信息传递](#服务器与客户端信息传递)
		- [Tag的数量容器](#tag的数量容器)
		- [网络同步变量生命周期申明](#网络同步变量生命周期申明)
		- [GAS部分](#gas部分)
			- [ASC 创建](#asc-创建)
			- [ASC初始化](#asc初始化)
			- [能力集初始化](#能力集初始化)
		- [角色的初始化再次推进](#角色的初始化再次推进)
	- [代码](#代码)
		- [LyraPlayState](#lyraplaystate)
		- [GameplayTagStack](#gameplaytagstack)
	- [总结](#总结)



## 概述
这节稍微麻烦点.原本计划时把ASC和队伍接口都剥离开完全不讲的.但是看了下里面有个FastArray必须要讲.所以这节还是直接把这个类大部分都搞定.
关于ASC和队伍系统的流程不在本节进行阐述.
## LyraPlayerState
### 区分玩家连接是否活跃
``` cpp
/** Defines the types of client connected */
/** 定义已连接客户端的类型 */
UENUM()
enum class ELyraPlayerConnectionType : uint8
{
	// An active player
	// 一名活跃的玩家
	Player = 0,

	// Spectator connected to a running game
	// 观众与正在进行的比赛相连接
	LiveSpectator,

	// Spectating a demo recording offline
	// 离线观看演示录制内容
	ReplaySpectator,

	// A deactivated player (disconnected)
	// 一个已停用的玩家（已断线）
	InactivePlayer
};

```
根据需要 我们可以在OnDeactivated决定是否需要掉线自动销毁PlayerState,或者做一个延时的等待.
``` cpp
	/** 当拥有该玩家状态的玩家断开连接时，服务器会调用此方法，通常情况下此方法会销毁该玩家状态 */
	// 计划添加到体验中 是否开启支持自动摧毁玩家状态
	UE_API virtual void OnDeactivated() override;
	
	/** 当拥有该玩家状态的玩家重新连接后，服务器会调用此函数，并将此玩家状态添加到活跃玩家数组中 */
	// 切换到玩家链接为活跃状态
	UE_API virtual void OnReactivated() override;

```

### 设置PawnData

``` cpp

	// 获取PawnData
	template <class T>
	const T* GetPawnData() const { return Cast<T>(PawnData); }

	// 在[服务器]得Experience加载完成之后,设置PawnData,然后通过网络同步到客户端
	UE_API void SetPawnData(const ULyraPawnData* InPawnData);

	// 客户端这个变量需要从网络同步过来
	UPROPERTY(ReplicatedUsing = OnRep_PawnData)
	TObjectPtr<const ULyraPawnData> PawnData;


```
这个PawnData主要是在OnExperienceLoaded通过GameMode获取从而设置.
``` cpp
void ALyraPlayerState::OnExperienceLoaded(const ULyraExperienceDefinition* /*CurrentExperience*/)
{
	// 服务器才有GameMode
	if (ALyraGameMode* LyraGameMode = GetWorld()->GetAuthGameMode<ALyraGameMode>())
	{
		// 通过控制器获取PawnData
		if (const ULyraPawnData* NewPawnData = LyraGameMode->GetPawnDataForController(GetOwningController()))
		{
			SetPawnData(NewPawnData);
		}
		else
		{
			UE_LOG(LogLyra, Error, TEXT("ALyraPlayerState::OnExperienceLoaded(): Unable to find PawnData to initialize player state [%s]!"), *GetNameSafe(this));
		}
	}
}

```
它是在PostInitializeComponents中进行绑定的
``` cpp
void ALyraPlayerState::PostInitializeComponents()
{
	Super::PostInitializeComponents();

	check(AbilitySystemComponent);
	// 初始化ASC组件
	// 逻辑实际拥有者 是PlayerState
	// 替身操作者 是控制得Pawn
	AbilitySystemComponent->InitAbilityActorInfo(this, GetPawn());

	UWorld* World = GetWorld();
	// 世界必须存在,网络模式不能是客户端,因为客户端需要由服务器属性同步过去
	if (World && World->IsGameWorld() && World->GetNetMode() != NM_Client)
	{
		AGameStateBase* GameState = GetWorld()->GetGameState();

		check(GameState);
		ULyraExperienceManagerComponent* ExperienceComponent = GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
		check(ExperienceComponent);
		// 绑定体验加载完成之后需要执行的函数
		ExperienceComponent->CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
	}
}
```
在这里完成初始化中之后,我们还需要修复GameMode中获取PawnData的方法,以便完成获取逻辑!
``` cpp
const ULyraPawnData* ALyraGameMode::GetPawnDataForController(const AController* InController) const
{
	// See if pawn data is already set on the player state
	// 检查是否已为玩家状态设置了棋子数据


	
	// 控制器不应该为空!
	if (InController != nullptr)
	{
		// AI控制器也会有PlayerState,通过其的构造函数开启
		if (const ALyraPlayerState* LyraPS = InController->GetPlayerState<ALyraPlayerState>())
		{
			//@XGTODO:我们现在还没有写PlayerState
			if (const ULyraPawnData* PawnData = LyraPS->GetPawnData<ULyraPawnData>())
			{
				return PawnData;
			}
		}
	}
	//...
}

```

### 设置队伍
此处仅简单提及.
我们需要先定义队伍接口
``` cpp
/** Interface for actors which can be associated with teams */
/** 用于表示可与团队相关联的演员的接口 */
UINTERFACE(MinimalAPI, meta=(CannotImplementInterfaceInBlueprint))
class ULyraTeamAgentInterface : public UGenericTeamAgentInterface
{
	GENERATED_UINTERFACE_BODY()
};

class ILyraTeamAgentInterface : public IGenericTeamAgentInterface
{
	GENERATED_IINTERFACE_BODY()

	// 获取队伍改变的代理
	virtual FOnLyraTeamIndexChangedDelegate* GetOnTeamIndexChangedDelegate() { return nullptr; }

	// 广播队伍发生了改变
	static UE_API void ConditionalBroadcastTeamChanged(TScriptInterface<ILyraTeamAgentInterface> This, FGenericTeamId OldTeamID, FGenericTeamId NewTeamID);

	// 带有检查的获取队伍改变的代理
	FOnLyraTeamIndexChangedDelegate& GetTeamChangedDelegateChecked()
	{
		FOnLyraTeamIndexChangedDelegate* Result = GetOnTeamIndexChangedDelegate();
		check(Result);
		return *Result;
	}
};

```
然后在PlayerState中实现
``` cpp
public:
	//~ILyraTeamAgentInterface interface
	// 设置队伍ID
	UE_API virtual void SetGenericTeamId(const FGenericTeamId& NewTeamID) override;
	// 获取队伍ID
	UE_API virtual FGenericTeamId GetGenericTeamId() const override;
	UE_API virtual FOnLyraTeamIndexChangedDelegate* GetOnTeamIndexChangedDelegate() override;
	//~End of ILyraTeamAgentInterface interface
private:
	// 队伍发生改变的代理
	UPROPERTY()
	FOnLyraTeamIndexChangedDelegate OnTeamChangedDelegate;

```
#### 设置队伍
其中设置队伍会在队伍创建组件中调用:
``` cpp
void ULyraTeamCreationComponent::OnExperienceLoaded(const ULyraExperienceDefinition* Experience)
{
#if WITH_SERVER_CODE
	if (HasAuthority())
	{
		ServerCreateTeams();
		ServerAssignPlayersToTeams();
	}
#endif
}



void ULyraTeamCreationComponent::ServerCreateTeams()
{
	for (const auto& KVP : TeamsToCreate)
	{
		const int32 TeamId = KVP.Key;
		ServerCreateTeam(TeamId, KVP.Value);
	}
}

void ULyraTeamCreationComponent::ServerAssignPlayersToTeams()
{
	// Assign players that already exist to teams
	AGameStateBase* GameState = GetGameStateChecked<AGameStateBase>();
	for (APlayerState* PS : GameState->PlayerArray)
	{
		if (ALyraPlayerState* LyraPS = Cast<ALyraPlayerState>(PS))
		{
			ServerChooseTeamForPlayer(LyraPS);
		}
	}

	// Listen for new players logging in
	ALyraGameMode* GameMode = Cast<ALyraGameMode>(GameState->AuthorityGameMode);
	check(GameMode);

	GameMode->OnGameModePlayerInitialized.AddUObject(this, &ThisClass::OnPlayerInitialized);
}

void ULyraTeamCreationComponent::ServerChooseTeamForPlayer(ALyraPlayerState* PS)
{
	if (PS->IsOnlyASpectator())
	{
		PS->SetGenericTeamId(FGenericTeamId::NoTeam);
	}
	else
	{
		const FGenericTeamId TeamID = IntegerToGenericTeamId(GetLeastPopulatedTeamID());
		PS->SetGenericTeamId(TeamID);
	}
}
```
#### 下级传递
其中传递方向为PlayerState->PlayerController->Charactar->蓝图


#### 队伍与战队ID
子战队ID目前没有设置额外的功能.
``` cpp
	// 队伍ID
	UPROPERTY(ReplicatedUsing=OnRep_MyTeamID)
	FGenericTeamId MyTeamID;

	// 子战队ID
	UPROPERTY(ReplicatedUsing=OnRep_MySquadID)
	int32 MySquadID;

```
### 观战角度的同步
``` cpp
	// Gets the replicated view rotation of this player, used for spectating
	// 获取此玩家的复制视图旋转角度，用于旁观模式使用
	UE_API FRotator GetReplicatedViewRotation() const;

	// Sets the replicated view rotation, only valid on the server
	// 设置复制视图的旋转角度，仅在服务器端有效 由控制器从PlayerTick中调用
	UE_API void SetReplicatedViewRotation(const FRotator& NewRotation);

	// 用于观战的同步角度
	UPROPERTY(Replicated)
	FRotator ReplicatedViewRotation;

```

### 服务器与客户端信息传递
``` cpp

	/*
	 * Send a message to just this player
	 * (use only for client notifications like accolades, quest toasts, etc... that can handle being occasionally lost)
	 * 向仅此一位玩家发送消息
	 * 仅用于客户端通知，例如荣誉奖励、任务庆祝信息等，这类信息偶尔丢失也是可以接受的）
	 *
	 * 服务器调用,客户端执行
	 */
	UFUNCTION(Client, Unreliable, BlueprintCallable, Category = "Lyra|PlayerState")
	UE_API void ClientBroadcastMessage(const FLyraVerbMessage Message);


```

### Tag的数量容器

这里同步的Tag通过FastArray进行实现.
FastArray的特性不在本节阐述.
该项目中多处用到FastArray.一定要掌握!!!!!
该容器的增删改查都由该FGameplayTagStackContainer结构体实现.拥有者转发即可!
``` cpp

	// Adds a specified number of stacks to the tag (does nothing if StackCount is below 1)
	// 向标签中添加指定数量的堆栈（若堆栈数量少于 1，则不执行任何操作）
	UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Teams)
	UE_API void AddStatTagStack(FGameplayTag Tag, int32 StackCount);

	// Removes a specified number of stacks from the tag (does nothing if StackCount is below 1)
	// 从标签中移除指定数量的堆栈（若“堆栈数量”小于 1，则不执行任何操作）
	UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Teams)
	UE_API void RemoveStatTagStack(FGameplayTag Tag, int32 StackCount);

	// Returns the stack count of the specified tag (or 0 if the tag is not present)
	// 返回指定标签的栈数量（若该标签不存在则返回 0）
	UFUNCTION(BlueprintCallable, Category=Teams)
	UE_API int32 GetStatTagStackCount(FGameplayTag Tag) const;

	// Returns true if there is at least one stack of the specified tag
	// 如果存在指定标签的至少一个栈，则返回 true
	UFUNCTION(BlueprintCallable, Category=Teams)
	UE_API bool HasStatTag(FGameplayTag Tag) const;

	// Tag的容器
	UPROPERTY(Replicated)
	FGameplayTagStackContainer StatTags;

```
### 网络同步变量生命周期申明
``` cpp
void ALyraPlayerState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	FDoRepLifetimeParams SharedParams;

	/** 此属性是否采用推送模型。请参阅 PushModel.h 文件 */
	SharedParams.bIsPushBased = true;

	DOREPLIFETIME_WITH_PARAMS_FAST(ThisClass, PawnData, SharedParams);
	DOREPLIFETIME_WITH_PARAMS_FAST(ThisClass, MyPlayerConnectionType, SharedParams)
	DOREPLIFETIME_WITH_PARAMS_FAST(ThisClass, MyTeamID, SharedParams);
	DOREPLIFETIME_WITH_PARAMS_FAST(ThisClass, MySquadID, SharedParams);

	// 跳过拥有者
	SharedParams.Condition = ELifetimeCondition::COND_SkipOwner;
	DOREPLIFETIME_WITH_PARAMS_FAST(ThisClass, ReplicatedViewRotation, SharedParams);

	DOREPLIFETIME(ThisClass, StatTags);	
}

```

### GAS部分
此处仅简单阐述.

#### ASC 创建
``` cpp
ALyraPlayerState::ALyraPlayerState(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
	, MyPlayerConnectionType(ELyraPlayerConnectionType::Player)
{
	AbilitySystemComponent = ObjectInitializer.CreateDefaultSubobject<ULyraAbilitySystemComponent>(this, TEXT("AbilitySystemComponent"));
	AbilitySystemComponent->SetIsReplicated(true);
	AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);

	//@XGTODO:这部分内容需要到GAS章节去写
	// These attribute sets will be detected by AbilitySystemComponent::InitializeComponent. Keeping a reference so that the sets don't get garbage collected before that.
	/*HealthSet = CreateDefaultSubobject<ULyraHealthSet>(TEXT("HealthSet"));
	CombatSet = CreateDefaultSubobject<ULyraCombatSet>(TEXT("CombatSet"));*/

	// AbilitySystemComponent needs to be updated at a high frequency.
	SetNetUpdateFrequency(100.0f);

	MyTeamID = FGenericTeamId::NoTeam;
	MySquadID = INDEX_NONE;
}

```
#### ASC初始化
在PostInitialComponents中进行ASC的初始化
``` cpp
void ALyraPlayerState::PostInitializeComponents()
{
	Super::PostInitializeComponents();

	check(AbilitySystemComponent);
	// 初始化ASC组件
	// 逻辑实际拥有者 是PlayerState
	// 替身操作者 是控制得Pawn
	AbilitySystemComponent->InitAbilityActorInfo(this, GetPawn());

	UWorld* World = GetWorld();
	// 世界必须存在,网络模式不能是客户端,因为客户端需要由服务器属性同步过去
	if (World && World->IsGameWorld() && World->GetNetMode() != NM_Client)
	{
		AGameStateBase* GameState = GetWorld()->GetGameState();

		check(GameState);
		ULyraExperienceManagerComponent* ExperienceComponent = GameState->FindComponentByClass<ULyraExperienceManagerComponent>();
		check(ExperienceComponent);
		// 绑定体验加载完成之后需要执行的函数
		ExperienceComponent->CallOrRegister_OnExperienceLoaded(FOnLyraExperienceLoaded::FDelegate::CreateUObject(this, &ThisClass::OnExperienceLoaded));
	}
}
```
#### 能力集初始化
``` cpp
void ALyraPlayerState::OnExperienceLoaded(const ULyraExperienceDefinition* /*CurrentExperience*/)
{
	// 服务器才有GameMode
	if (ALyraGameMode* LyraGameMode = GetWorld()->GetAuthGameMode<ALyraGameMode>())
	{
		// 通过控制器获取PawnData
		if (const ULyraPawnData* NewPawnData = LyraGameMode->GetPawnDataForController(GetOwningController()))
		{
			SetPawnData(NewPawnData);
		}
		else
		{
			UE_LOG(LogLyra, Error, TEXT("ALyraPlayerState::OnExperienceLoaded(): Unable to find PawnData to initialize player state [%s]!"), *GetNameSafe(this));
		}
	}
}


```
``` cpp
void ALyraPlayerState::SetPawnData(const ULyraPawnData* InPawnData)
{

	// 输入的PawnData必须有效
	check(InPawnData);

	// 这个角色必须具有权威性 否则不生效
	if (GetLocalRole() != ROLE_Authority)
	{
		return;
	}

	if (PawnData)
	{
		UE_LOG(LogLyra, Error, TEXT("Trying to set PawnData [%s] on player state [%s] that already has valid PawnData [%s]."), *GetNameSafe(InPawnData), *GetNameSafe(this), *GetNameSafe(PawnData));
		return;
	}

	// 标记数据为脏
	MARK_PROPERTY_DIRTY_FROM_NAME(ThisClass, PawnData, this);
	PawnData = InPawnData;

	// 通过PawnData 注册能力集
	//@XGTODO: 在讲GAS前 这块需要注释掉
	/*for (const ULyraAbilitySet* AbilitySet : PawnData->AbilitySets)
	{
		if (AbilitySet)
		{
			AbilitySet->GiveToAbilitySystem(AbilitySystemComponent, nullptr);
		}
	}*/
	// 发送框架事件 GAS注册完毕!
	UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(this, NAME_LyraAbilityReady);
	
	/** 强制将角色信息更新至客户端/演示网络驱动程序 */
	ForceNetUpdate();
}

```

### 角色的初始化再次推进
这里是为了解决人物角色在网络初始化过程中的,多次推进问题的其中一次推进.
现在不要关心.明白即可.
``` cpp
void ALyraPlayerState::ClientInitialize(AController* C)
{
	Super::ClientInitialize(C);

	if (ULyraPawnExtensionComponent* PawnExtComp = ULyraPawnExtensionComponent::FindPawnExtensionComponent(GetPawn()))
	{
		PawnExtComp->CheckDefaultInitialization();
	}
}
```

## 代码

### LyraPlayState
``` cpp


/** Defines the types of client connected */
/** 定义已连接客户端的类型 */
UENUM()
enum class ELyraPlayerConnectionType : uint8
{
	// An active player
	// 一名活跃的玩家
	Player = 0,

	// Spectator connected to a running game
	// 观众与正在进行的比赛相连接
	LiveSpectator,

	// Spectating a demo recording offline
	// 离线观看演示录制内容
	ReplaySpectator,

	// A deactivated player (disconnected)
	// 一个已停用的玩家（已断线）
	InactivePlayer
};

/**
 * ALyraPlayerState
 *
 *	Base player state class used by this project.
 *	本项目所使用的基础玩家状态类。
 *	
 */
UCLASS(MinimalAPI, Config = Game)
class ALyraPlayerState : public AModularPlayerState, public IAbilitySystemInterface, public ILyraTeamAgentInterface
{
	GENERATED_BODY()

public:
	/*
	 * 构造函数
	 * 创建GAS组件 开启网路同步 设置同步模式
	 * 创建生命值组件
	 * 创建战斗组件
	 * 设置同步率
	 * 初始化队伍信息
	 */
	UE_API ALyraPlayerState(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	// 获取玩家控制器
	UFUNCTION(BlueprintCallable, Category = "Lyra|PlayerState")
	UE_API ALyraPlayerController* GetLyraPlayerController() const;

	// 获取 LyraASC 该组件通过构造函数创建
	UFUNCTION(BlueprintCallable, Category = "Lyra|PlayerState")
	ULyraAbilitySystemComponent* GetLyraAbilitySystemComponent() const { return AbilitySystemComponent; }
	// 这是IAbilitySystemInterface的重写方法 返回成员变量即可
	UE_API virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

	// 获取PawnData
	template <class T>
	const T* GetPawnData() const { return Cast<T>(PawnData); }

	// 在[服务器]得Experience加载完成之后,设置PawnData,然后通过网络同步到客户端
	UE_API void SetPawnData(const ULyraPawnData* InPawnData);

	//~AActor interface
	// 执行父类 暂无作用
	UE_API virtual void PreInitializeComponents() override;
	// 执行父类 初始化ASC的ActorInfo 然后绑定体验加载完成回调
	UE_API virtual void PostInitializeComponents() override;
	//~End of AActor interface

	//~APlayerState interface
	
	/** 将角色重置至初始状态 - 用于在不重新加载的情况下重新开始关卡时使用。*/
	UE_API virtual void Reset() override;
	
	/** 当控制器的玩家状态首次进行复制时，会调用此方法。*/
	UE_API virtual void ClientInitialize(AController* C) override;
	
	/** 复制需要保存在非活动玩家状态中的属性 */
	UE_API virtual void CopyProperties(APlayerState* PlayerState) override;

	
	/** 当拥有该玩家状态的玩家断开连接时，服务器会调用此方法，通常情况下此方法会销毁该玩家状态 */
	// 计划添加到体验中 是否开启支持自动摧毁玩家状态
	UE_API virtual void OnDeactivated() override;
	
	/** 当拥有该玩家状态的玩家重新连接后，服务器会调用此函数，并将此玩家状态添加到活跃玩家数组中 */
	// 切换到玩家链接为活跃状态
	UE_API virtual void OnReactivated() override;
	
	//~End of APlayerState interface

	//~ILyraTeamAgentInterface interface
	// 设置队伍ID
	UE_API virtual void SetGenericTeamId(const FGenericTeamId& NewTeamID) override;
	// 获取队伍ID
	UE_API virtual FGenericTeamId GetGenericTeamId() const override;
	UE_API virtual FOnLyraTeamIndexChangedDelegate* GetOnTeamIndexChangedDelegate() override;
	//~End of ILyraTeamAgentInterface interface

	static UE_API const FName NAME_LyraAbilityReady;

	// 设置玩家链接类型
	UE_API void SetPlayerConnectionType(ELyraPlayerConnectionType NewType);
	// 获取玩家链接类型
	ELyraPlayerConnectionType GetPlayerConnectionType() const { return MyPlayerConnectionType; }

	/** Returns the Squad ID of the squad the player belongs to. */
	/** 返回玩家所属小队的编号。*/
	UFUNCTION(BlueprintCallable)
	int32 GetSquadId() const
	{
		return MySquadID;
	}

	/** Returns the Team ID of the team the player belongs to. */
	/** 返回玩家所属团队的团队 ID 。*/
	UFUNCTION(BlueprintCallable)
	int32 GetTeamId() const
	{
		return GenericTeamIdToInteger(MyTeamID);
	}
	// 设置小队编号
	UE_API void SetSquadID(int32 NewSquadID);

	// Adds a specified number of stacks to the tag (does nothing if StackCount is below 1)
	// 向标签中添加指定数量的堆栈（若堆栈数量少于 1，则不执行任何操作）
	UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Teams)
	UE_API void AddStatTagStack(FGameplayTag Tag, int32 StackCount);

	// Removes a specified number of stacks from the tag (does nothing if StackCount is below 1)
	// 从标签中移除指定数量的堆栈（若“堆栈数量”小于 1，则不执行任何操作）
	UFUNCTION(BlueprintCallable, BlueprintAuthorityOnly, Category=Teams)
	UE_API void RemoveStatTagStack(FGameplayTag Tag, int32 StackCount);

	// Returns the stack count of the specified tag (or 0 if the tag is not present)
	// 返回指定标签的栈数量（若该标签不存在则返回 0）
	UFUNCTION(BlueprintCallable, Category=Teams)
	UE_API int32 GetStatTagStackCount(FGameplayTag Tag) const;

	// Returns true if there is at least one stack of the specified tag
	// 如果存在指定标签的至少一个栈，则返回 true
	UFUNCTION(BlueprintCallable, Category=Teams)
	UE_API bool HasStatTag(FGameplayTag Tag) const;


	/*
	 * Send a message to just this player
	 * (use only for client notifications like accolades, quest toasts, etc... that can handle being occasionally lost)
	 * 向仅此一位玩家发送消息
	 * 仅用于客户端通知，例如荣誉奖励、任务庆祝信息等，这类信息偶尔丢失也是可以接受的）
	 *
	 * 服务器调用,客户端执行
	 */
	UFUNCTION(Client, Unreliable, BlueprintCallable, Category = "Lyra|PlayerState")
	UE_API void ClientBroadcastMessage(const FLyraVerbMessage Message);

	// Gets the replicated view rotation of this player, used for spectating
	// 获取此玩家的复制视图旋转角度，用于旁观模式使用
	UE_API FRotator GetReplicatedViewRotation() const;

	// Sets the replicated view rotation, only valid on the server
	// 设置复制视图的旋转角度，仅在服务器端有效
	UE_API void SetReplicatedViewRotation(const FRotator& NewRotation);

private:
	/*
	 * 在初始化的过程中绑定执行等待体验加载完毕后,读取设置PawnData
	 * PostInitializeComponents中绑定
	 * 
	 */
	UE_API void OnExperienceLoaded(const ULyraExperienceDefinition* CurrentExperience);

protected:

	// 暂无功能
	UFUNCTION()
	UE_API void OnRep_PawnData();

protected:

	// 客户端这个变量需要从网络同步过来
	UPROPERTY(ReplicatedUsing = OnRep_PawnData)
	TObjectPtr<const ULyraPawnData> PawnData;

private:

	// The ability system component sub-object used by player characters.
	// 用于玩家角色的“能力系统”子组件。
	UPROPERTY(VisibleAnywhere, Category = "Lyra|PlayerState")
	TObjectPtr<ULyraAbilitySystemComponent> AbilitySystemComponent;

	// Health attribute set used by this actor.
	// 该角色所使用的健康属性设定。
	UPROPERTY()
	TObjectPtr<const class ULyraHealthSet> HealthSet;
	
	// Combat attribute set used by this actor.
	// 此角色所使用的战斗属性设定。
	UPROPERTY()
	TObjectPtr<const class ULyraCombatSet> CombatSet;

	// 玩家链接的类型需要进行网络同步
	UPROPERTY(Replicated)
	ELyraPlayerConnectionType MyPlayerConnectionType;

	// 队伍发生改变的代理
	UPROPERTY()
	FOnLyraTeamIndexChangedDelegate OnTeamChangedDelegate;

	// 队伍ID
	UPROPERTY(ReplicatedUsing=OnRep_MyTeamID)
	FGenericTeamId MyTeamID;

	// 子战队ID
	UPROPERTY(ReplicatedUsing=OnRep_MySquadID)
	int32 MySquadID;

	// Tag的容器
	UPROPERTY(Replicated)
	FGameplayTagStackContainer StatTags;

	// 用于观战的同步角度
	UPROPERTY(Replicated)
	FRotator ReplicatedViewRotation;

private:
	// 通知队伍发生了改变
	UFUNCTION()
	UE_API void OnRep_MyTeamID(FGenericTeamId OldTeamID);

	// 暂无功能
	UFUNCTION()
	UE_API void OnRep_MySquadID();
};

```

### GameplayTagStack

``` cpp

/**
 * Represents one stack of a gameplay tag (tag + count)
 * 表示一个游戏玩法标签的一组数据（标签 + 数量）
 */
USTRUCT(BlueprintType)
struct FGameplayTagStack : public FFastArraySerializerItem
{
	GENERATED_BODY()

	FGameplayTagStack()
	{}

	FGameplayTagStack(FGameplayTag InTag, int32 InStackCount)
		: Tag(InTag)
		, StackCount(InStackCount)
	{
	}

	FString GetDebugString() const;

private:
	friend FGameplayTagStackContainer;

	UPROPERTY()
	FGameplayTag Tag;

	UPROPERTY()
	int32 StackCount = 0;
};

/** Container of gameplay tag stacks */
/** 游戏玩法标签堆栈的容器 */
USTRUCT(BlueprintType)
struct FGameplayTagStackContainer : public FFastArraySerializer
{
	GENERATED_BODY()

	FGameplayTagStackContainer()
	//	: Owner(nullptr)
	{
	}

public:
	// Adds a specified number of stacks to the tag (does nothing if StackCount is below 1)
	// 向标签中添加指定数量的堆栈（若堆栈数量少于 1，则不执行任何操作）
	void AddStack(FGameplayTag Tag, int32 StackCount);

	// Removes a specified number of stacks from the tag (does nothing if StackCount is below 1)
	// 从标签中移除指定数量的堆栈（若“堆栈数量”小于 1，则不执行任何操作）
	void RemoveStack(FGameplayTag Tag, int32 StackCount);

	// Returns the stack count of the specified tag (or 0 if the tag is not present)
	// 返回指定标签的栈数量（若该标签不存在则返回 0）
	int32 GetStackCount(FGameplayTag Tag) const
	{
		return TagToCountMap.FindRef(Tag);
	}

	// Returns true if there is at least one stack of the specified tag
	// 如果存在指定标签的至少一个栈，则返回 true
	bool ContainsTag(FGameplayTag Tag) const
	{
		return TagToCountMap.Contains(Tag);
	}

	//~FFastArraySerializer contract
	void PreReplicatedRemove(const TArrayView<int32> RemovedIndices, int32 FinalSize);
	void PostReplicatedAdd(const TArrayView<int32> AddedIndices, int32 FinalSize);
	void PostReplicatedChange(const TArrayView<int32> ChangedIndices, int32 FinalSize);
	//~End of FFastArraySerializer contract

	bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms)
	{
		return FFastArraySerializer::FastArrayDeltaSerialize<FGameplayTagStack, FGameplayTagStackContainer>(Stacks, DeltaParms, *this);
	}

private:
	// Replicated list of gameplay tag stacks
	// 游戏玩法标签堆叠的复制列表
	UPROPERTY()
	TArray<FGameplayTagStack> Stacks;
	
	// Accelerated list of tag stacks for queries
	// 查询用标签栈的加速列表
	TMap<FGameplayTag, int32> TagToCountMap;
};

template<>
struct TStructOpsTypeTraits<FGameplayTagStackContainer> : public TStructOpsTypeTraitsBase2<FGameplayTagStackContainer>
{
	enum
	{
		WithNetDeltaSerializer = true,
	};
};


```

## 总结
本节详细介绍了PlayerState的业务功能.