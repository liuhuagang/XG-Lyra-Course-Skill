# UE5_Lyra学习指南_065_人物基类

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_065\_人物基类](#ue5_lyra学习指南_065_人物基类)
	- [概述](#概述)
	- [LyraPawn](#lyrapawn)
		- [代码](#代码)
		- [Possessed](#possessed)
		- [UnPossessed](#unpossessed)
		- [修改队伍ID](#修改队伍id)
	- [LyraCharacterWithAbilities](#lyracharacterwithabilities)
		- [代码](#代码-1)
	- [LyraCharacter](#lyracharacter)
		- [父类拓展接口](#父类拓展接口)
			- [AModularCharacter](#amodularcharacter)
			- [IAbilitySystemInterface](#iabilitysysteminterface)
			- [IGameplayCueInterface](#igameplaycueinterface)
			- [IGameplayTagAssetInterface](#igameplaytagassetinterface)
			- [ILyraTeamAgentInterface](#ilyrateamagentinterface)
		- [构造函数](#构造函数)
		- [获取游戏玩法类](#获取游戏玩法类)
		- [ASC获取](#asc获取)
		- [Tag的转发获取](#tag的转发获取)
		- [重要度管理器](#重要度管理器)
		- [队伍切换](#队伍切换)
		- [加速度同步](#加速度同步)
			- [传输结构体定义](#传输结构体定义)
			- [笛卡尔左边转换为极坐标](#笛卡尔左边转换为极坐标)
			- [属性同步到客户端传递给移动组件](#属性同步到客户端传递给移动组件)
		- [移动数据同步](#移动数据同步)
			- [结构体定义](#结构体定义)
			- [LyraReplicationGraph调用UpdateSharedReplication](#lyrareplicationgraph调用updatesharedreplication)
			- [服务端计算数据](#服务端计算数据)
			- [客户端更新数据](#客户端更新数据)
		- [业务转发给生命值组件](#业务转发给生命值组件)
			- [绑定属性](#绑定属性)
			- [死亡流程](#死亡流程)
			- [掉出边界](#掉出边界)
		- [业务转发给PawnExt](#业务转发给pawnext)
		- [移动tag的处理](#移动tag的处理)
			- [移动枚举](#移动枚举)
			- [移动Tag](#移动tag)
			- [绑定转换](#绑定转换)
		- [蹲伏和跳跃](#蹲伏和跳跃)
		- [代码](#代码-2)
	- [总结](#总结)



## 概述
本节主要讲解LyraCharacter的代码作用.
它是我们项目的人物基类.主要起了中转的作用.涉及到ASC,网络同步,相机模式等具体转发的业务内容不在本节详细论述.


## LyraPawn
未在本项目中使用.直接给出代码即可.
关于队伍信息从控制器传播过来,同时由Pawn传播出去,不再赘述.

### 代码
``` cpp

/**
 * ALyraPawn
 */
UCLASS(MinimalAPI)
class ALyraPawn : public AModularPawn, public ILyraTeamAgentInterface
{
	GENERATED_BODY()

public:
	// 无
	UE_API ALyraPawn(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	//~AActor interface
	// 无
	UE_API virtual void PreInitializeComponents() override;
	// 无
	UE_API virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
	//~End of AActor interface

	//~APawn interface
	// 绑定和传递队伍变更信息
	UE_API virtual void PossessedBy(AController* NewController) override;
	// 停止绑定和是否恢复上一次的队伍信息
	UE_API virtual void UnPossessed() override;
	//~End of APawn interface

	//~ILyraTeamAgentInterface interface
	// 设置队伍信息 注意区分是否有网络权限和设置权限
	UE_API virtual void SetGenericTeamId(const FGenericTeamId& NewTeamID) override;
	// 获取队伍信息
	UE_API virtual FGenericTeamId GetGenericTeamId() const override;
	// 获取绑定的代理,方便向下级传递
	UE_API virtual FOnLyraTeamIndexChangedDelegate* GetOnTeamIndexChangedDelegate() override;
	//~End of ILyraTeamAgentInterface interface

protected:
	// Called to determine what happens to the team ID when possession ends
	// 用于确定当控球权结束时团队标识将如何处理的函数
	virtual FGenericTeamId DetermineNewTeamAfterPossessionEnds(FGenericTeamId OldTeamID) const
	{
		// This could be changed to return, e.g., OldTeamID if you want to keep it assigned afterwards, or return an ID for some neutral faction, or etc...
		// 这可以修改为返回诸如“OldTeamID”之类的值，这样您就可以在之后继续保留该团队的标识，或者返回某个中立阵营的标识，或者等等。
		return FGenericTeamId::NoTeam;
	}

private:
	// 接收上级队伍信息的变更传递
	UFUNCTION()
	UE_API void OnControllerChangedTeam(UObject* TeamAgent, int32 OldTeam, int32 NewTeam);

private:
	// 队伍ID
	UPROPERTY(ReplicatedUsing = OnRep_MyTeamID)
	FGenericTeamId MyTeamID;

	// 队伍变更的代理
	UPROPERTY()
	FOnLyraTeamIndexChangedDelegate OnTeamChangedDelegate;

private:
	// 属性同步后的队伍信息变更函数
	UFUNCTION()
	UE_API void OnRep_MyTeamID(FGenericTeamId OldTeamID);
};
```
### Possessed
当Pawn受到控制时,绑定到控制器的队伍变更.同时触发一次队伍变更传递
``` cpp
void ALyraPawn::PossessedBy(AController* NewController)
{
	const FGenericTeamId OldTeamID = MyTeamID;

	Super::PossessedBy(NewController);

	// Grab the current team ID and listen for future changes
	// 获取当前团队的 ID，并监听未来的变化情况
	if (ILyraTeamAgentInterface* ControllerAsTeamProvider = Cast<ILyraTeamAgentInterface>(NewController))
	{
		MyTeamID = ControllerAsTeamProvider->GetGenericTeamId();
		ControllerAsTeamProvider->GetTeamChangedDelegateChecked().AddDynamic(this, &ThisClass::OnControllerChangedTeam);
	}
	ConditionalBroadcastTeamChanged(this, OldTeamID, MyTeamID);
}

```
### UnPossessed
当Pawn失去控制器时,取消绑定,同时再触发一次队伍ID变更传递.
此时可以决定是否恢复上次的队伍ID,或者直接给一个默认队伍ID或者无ID.
``` cpp

void ALyraPawn::UnPossessed()
{
	AController* const OldController = GetController();

	// Stop listening for changes from the old controller
	// 停止监听旧控制器的更改情况
	const FGenericTeamId OldTeamID = MyTeamID;
	if (ILyraTeamAgentInterface* ControllerAsTeamProvider = Cast<ILyraTeamAgentInterface>(OldController))
	{
		ControllerAsTeamProvider->GetTeamChangedDelegateChecked().RemoveAll(this);
	}

	Super::UnPossessed();

	// Determine what the new team ID should be afterwards
	// 接下来确定新的团队编号应该是多少
	MyTeamID = DetermineNewTeamAfterPossessionEnds(OldTeamID);
	ConditionalBroadcastTeamChanged(this, OldTeamID, MyTeamID);
}


```
```cpp
	// Called to determine what happens to the team ID when possession ends
	// 用于确定当控球权结束时团队标识将如何处理的函数
	virtual FGenericTeamId DetermineNewTeamAfterPossessionEnds(FGenericTeamId OldTeamID) const
	{
		// This could be changed to return, e.g., OldTeamID if you want to keep it assigned afterwards, or return an ID for some neutral faction, or etc...
		// 这可以修改为返回诸如“OldTeamID”之类的值，这样您就可以在之后继续保留该团队的标识，或者返回某个中立阵营的标识，或者等等。
		return FGenericTeamId::NoTeam;
	}


```
### 修改队伍ID
注意修改队伍ID只能在服务器上操作.并且要考虑是否受到控制器的约束.
如果有控制器,那么我们不能直接修改队伍信息,应当由控制前传递过来.
``` cpp
private:
	// 队伍ID
	UPROPERTY(ReplicatedUsing = OnRep_MyTeamID)
	FGenericTeamId MyTeamID;

	// 队伍变更的代理
	UPROPERTY()
	FOnLyraTeamIndexChangedDelegate OnTeamChangedDelegate;
```
``` cpp

void ALyraPawn::SetGenericTeamId(const FGenericTeamId& NewTeamID)
{
	if (GetController() == nullptr)
	{
		if (HasAuthority())
		{
			const FGenericTeamId OldTeamID = MyTeamID;
			MyTeamID = NewTeamID;
			ConditionalBroadcastTeamChanged(this, OldTeamID, MyTeamID);
		}
		else
		{
			UE_LOG(LogLyraTeams, Error, TEXT("You can't set the team ID on a pawn (%s) except on the authority"), *GetPathNameSafe(this));
		}
	}
	else
	{
		UE_LOG(LogLyraTeams, Error, TEXT("You can't set the team ID on a possessed pawn (%s); it's driven by the associated controller"), *GetPathNameSafe(this));
	}
}

FGenericTeamId ALyraPawn::GetGenericTeamId() const
{
	return MyTeamID;
}

FOnLyraTeamIndexChangedDelegate* ALyraPawn::GetOnTeamIndexChangedDelegate()
{
	return &OnTeamChangedDelegate;
}

void ALyraPawn::OnControllerChangedTeam(UObject* TeamAgent, int32 OldTeam, int32 NewTeam)
{
	const FGenericTeamId MyOldTeamID = MyTeamID;
	MyTeamID = IntegerToGenericTeamId(NewTeam);
	ConditionalBroadcastTeamChanged(this, MyOldTeamID, MyTeamID);
}

void ALyraPawn::OnRep_MyTeamID(FGenericTeamId OldTeamID)
{
	ConditionalBroadcastTeamChanged(this, OldTeamID, MyTeamID);
}

```
``` cpp
void ILyraTeamAgentInterface::ConditionalBroadcastTeamChanged(TScriptInterface<ILyraTeamAgentInterface> This, FGenericTeamId OldTeamID, FGenericTeamId NewTeamID)
{
	if (OldTeamID != NewTeamID)
	{
		const int32 OldTeamIndex = GenericTeamIdToInteger(OldTeamID); 
		const int32 NewTeamIndex = GenericTeamIdToInteger(NewTeamID);

		UObject* ThisObj = This.GetObject();
		UE_LOG(LogLyraTeams, Verbose, TEXT("[%s] %s assigned team %d"), *GetClientServerContextString(ThisObj), *GetPathNameSafe(ThisObj), NewTeamIndex);

		This.GetInterface()->GetTeamChangedDelegateChecked().Broadcast(ThisObj, OldTeamIndex, NewTeamIndex);
	}
}

```



## LyraCharacterWithAbilities
该类是继承自LyraCharacter的角色类,特殊点在于,它本身拥有ASC.
在本项目中未使用.
### 代码
``` cpp

// ALyraCharacter typically gets the ability system component from the possessing player state
// This represents a character with a self-contained ability system component.
// 阿莉拉角色通常会从拥有者的玩家状态中获取能力系统组件
// 这表示该角色拥有独立的能力系统组件。
UCLASS(MinimalAPI, Blueprintable)
class ALyraCharacterWithAbilities : public ALyraCharacter
{
	GENERATED_BODY()

public:
	// 创建成员变量
	UE_API ALyraCharacterWithAbilities(const FObjectInitializer& ObjectInitializer);

	// 初始化ASC
	UE_API virtual void PostInitializeComponents() override;
	// 获取ASC
	UE_API virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

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
};


```
``` cpp
ALyraCharacterWithAbilities::ALyraCharacterWithAbilities(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
{
	// 创建ASC
	AbilitySystemComponent = ObjectInitializer.CreateDefaultSubobject<ULyraAbilitySystemComponent>(this, TEXT("AbilitySystemComponent"));
	AbilitySystemComponent->SetIsReplicated(true);
	AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);

	// These attribute sets will be detected by AbilitySystemComponent::InitializeComponent. Keeping a reference so that the sets don't get garbage collected before that.
	// 这些属性集将由 AbilitySystemComponent::InitializeComponent 方法进行检测。为此保留一个引用，以确保这些属性集在该方法执行之前不会被垃圾回收。
	HealthSet = CreateDefaultSubobject<ULyraHealthSet>(TEXT("HealthSet"));
	CombatSet = CreateDefaultSubobject<ULyraCombatSet>(TEXT("CombatSet"));

	// AbilitySystemComponent needs to be updated at a high frequency.
	// 能力系统组件需要以较高的频率进行更新。
	SetNetUpdateFrequency(100.0f);
}

void ALyraCharacterWithAbilities::PostInitializeComponents()
{
	Super::PostInitializeComponents();

	check(AbilitySystemComponent);
	// 初始化能力演员信息
	AbilitySystemComponent->InitAbilityActorInfo(this, this);
}
```

## LyraCharacter
LyraCharacter极其重要!!!!
它实现了四个接口.
IAbilitySystemInterface.
IGameplayCueInterface.
IGameplayTagAssetInterface
ILyraTeamAgentInterface.
它的父类是AModularCharacter.
### 父类拓展接口
#### AModularCharacter
``` cpp
/** Minimal class that supports extension by game feature plugins */
/** 最基本的类，支持通过游戏功能插件进行扩展 */
UCLASS(MinimalAPI, Blueprintable)
class AModularCharacter : public ACharacter
{
	GENERATED_BODY()

public:
	//~ Begin AActor Interface
	UE_API virtual void PreInitializeComponents() override;
	UE_API virtual void BeginPlay() override;
	UE_API virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
	//~ End AActor Interface
};

```

``` cpp
void AModularCharacter::PreInitializeComponents()
{
	Super::PreInitializeComponents();

	UGameFrameworkComponentManager::AddGameFrameworkComponentReceiver(this);
}

void AModularCharacter::BeginPlay()
{
	UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(this, UGameFrameworkComponentManager::NAME_GameActorReady);

	Super::BeginPlay();
}

void AModularCharacter::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
	UGameFrameworkComponentManager::RemoveGameFrameworkComponentReceiver(this);

	Super::EndPlay(EndPlayReason);
}


```

#### IAbilitySystemInterface
``` cpp

/** Interface for actors that expose access to an ability system component */
UINTERFACE(MinimalAPI, meta = (CannotImplementInterfaceInBlueprint))
class UAbilitySystemInterface : public UInterface
{
	GENERATED_UINTERFACE_BODY()
};

class IAbilitySystemInterface
{
	GENERATED_IINTERFACE_BODY()

	/** Returns the ability system component to use for this actor. It may live on another actor, such as a Pawn using the PlayerState's component */
	virtual UAbilitySystemComponent* GetAbilitySystemComponent() const = 0;
};
```
#### IGameplayCueInterface
``` cpp

/** Interface for actors that wish to handle GameplayCue events from GameplayEffects. Native only because blueprints can't implement interfaces with native functions */
UINTERFACE(MinimalAPI, meta = (CannotImplementInterfaceInBlueprint))
class UGameplayCueInterface: public UInterface
{
	GENERATED_UINTERFACE_BODY()
};

class IGameplayCueInterface
{
	GENERATED_IINTERFACE_BODY()

public:

	/** Handle a single gameplay cue */
	UE_API virtual void HandleGameplayCue(UObject* Self, FGameplayTag GameplayCueTag, EGameplayCueEvent::Type EventType, const FGameplayCueParameters& Parameters);

	/** Wrapper that handles multiple cues */
	UE_API virtual void HandleGameplayCues(UObject* Self, const FGameplayTagContainer& GameplayCueTags, EGameplayCueEvent::Type EventType, const FGameplayCueParameters& Parameters);

	/**
	* Returns true if the object can currently accept gameplay cues associated with the given tag. Returns true by default.
	* Allows objects to opt out of cues in cases such as pending death
	*/
	UE_API virtual bool ShouldAcceptGameplayCue(UObject* Self, FGameplayTag GameplayCueTag, EGameplayCueEvent::Type EventType, const FGameplayCueParameters& Parameters);


	// DEPRECATED - use the UObject* signatures above

	/** Handle a single gameplay cue */
	UE_API virtual void HandleGameplayCue(AActor *Self, FGameplayTag GameplayCueTag, EGameplayCueEvent::Type EventType, const FGameplayCueParameters& Parameters);

	/** Wrapper that handles multiple cues */
	UE_API virtual void HandleGameplayCues(AActor *Self, const FGameplayTagContainer& GameplayCueTags, EGameplayCueEvent::Type EventType, const FGameplayCueParameters& Parameters);

	/** Returns true if the actor can currently accept gameplay cues associated with the given tag. Returns true by default. Allows actors to opt out of cues in cases such as pending death */
	UE_API virtual bool ShouldAcceptGameplayCue(AActor *Self, FGameplayTag GameplayCueTag, EGameplayCueEvent::Type EventType, const FGameplayCueParameters& Parameters);

	// END DEPRECATED


	/** Return the cue sets used by this object. This is optional and it is possible to leave this list empty. */
	virtual void GetGameplayCueSets(TArray<class UGameplayCueSet*>& OutSets) const {}

	/** Default native handler, called if no tag matches found */
	UE_API virtual void GameplayCueDefaultHandler(EGameplayCueEvent::Type EventType, const FGameplayCueParameters& Parameters);

	/** Internal function to map ufunctions directly to gameplaycue tags */
	UFUNCTION(BlueprintImplementableEvent, BlueprintCosmetic, Category = GameplayCue, meta = (BlueprintInternalUseOnly = "true"))
	UE_API void BlueprintCustomHandler(EGameplayCueEvent::Type EventType, const FGameplayCueParameters& Parameters);

	/** Call from a Cue handler event to continue checking for additional, more generic handlers. Called from the ability system blueprint library */
	UFUNCTION(BlueprintCallable, BlueprintCosmetic, Category="Ability|GameplayCue")
	UE_API virtual void ForwardGameplayCueToParent();

	/** Calls the UFunction override for a specific gameplay cue */
	static UE_API void DispatchBlueprintCustomHandler(UObject* Object, UFunction* Func, EGameplayCueEvent::Type EventType, const FGameplayCueParameters& Parameters);

	/** Clears internal cache of what classes implement which functions */
	static UE_API void ClearTagToFunctionMap();

	IGameplayCueInterface() : bForwardToParent(false) {}

private:
	/** If true, keep checking for additional handlers */
	bool bForwardToParent;
};

```
#### IGameplayTagAssetInterface
``` cpp
/** Interface for assets which contain gameplay tags */
UINTERFACE(BlueprintType, MinimalAPI, meta=(CannotImplementInterfaceInBlueprint))
class UGameplayTagAssetInterface : public UInterface
{
	GENERATED_UINTERFACE_BODY()
};

class IGameplayTagAssetInterface
{
	GENERATED_IINTERFACE_BODY()

	/**
	 * Get any owned gameplay tags on the asset
	 * 
	 * @param OutTags	[OUT] Set of tags on the asset
	 */
	virtual void GetOwnedGameplayTags(FGameplayTagContainer& TagContainer) const=0;

	/**
	 * Check if the asset has a gameplay tag that matches against the specified tag (expands to include parents of asset tags)
	 * 
	 * @param TagToCheck	Tag to check for a match
	 * 
	 * @return True if the asset has a gameplay tag that matches, false if not
	 */
	UFUNCTION(BlueprintCallable, Category=GameplayTags)
	GAMEPLAYTAGS_API virtual bool HasMatchingGameplayTag(FGameplayTag TagToCheck) const;

	/**
	 * Check if the asset has gameplay tags that matches against all of the specified tags (expands to include parents of asset tags)
	 * 
	 * @param TagContainer			Tag container to check for a match
	 * 
	 * @return True if the asset has matches all of the gameplay tags, will be true if container is empty
	 */
	UFUNCTION(BlueprintCallable, Category=GameplayTags)
	GAMEPLAYTAGS_API virtual bool HasAllMatchingGameplayTags(const FGameplayTagContainer& TagContainer) const;

	/**
	 * Check if the asset has gameplay tags that matches against any of the specified tags (expands to include parents of asset tags)
	 * 
	 * @param TagContainer			Tag container to check for a match
	 * 
	 * @return True if the asset has matches any of the gameplay tags, will be false if container is empty
	 */
	UFUNCTION(BlueprintCallable, Category=GameplayTags)
	GAMEPLAYTAGS_API virtual bool HasAnyMatchingGameplayTags(const FGameplayTagContainer& TagContainer) const;

protected:

	/**
	 * Gets the owned gameplay tags for the asset.  Exposed to allow redirects of existing GetOwnedGameplayTags calls.  In Blueprints, new nodes will use BlueprintGameplayTagLibrary's version.
	 */
	UFUNCTION(BlueprintCallable, Category = GameplayTags, BlueprintInternalUseOnly, meta=(DisplayName="Get Owned Gameplay Tags", AllowPrivateAccess=true))
	GAMEPLAYTAGS_API virtual UPARAM(DisplayName = "Owned Tags") FGameplayTagContainer BP_GetOwnedGameplayTags() const;
};


```

#### ILyraTeamAgentInterface
``` cpp
/** Interface for actors which can be associated with teams */
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
### 构造函数
``` cpp
	/**
	 * 构造函数 非常重要
	 * 修改胶囊体组件的默认参数和碰撞预设
	 * 修改网格体组件的默认参数和碰撞预设
	 * 修改移动组件的默认类和默认参数
	 *
	 * 创建角色拓展组件,绑定ASC初始化和移除时的代理,用于出发生命值组件去绑定属性,用于Tag的接口初始化
	 * 创建生命值组件,用于接收ASC属性的绑定,并接收关于死亡流程的触发和完成
	 * 创建相机组件,设置相对位置
	 *
	 * 修改角色的默认参数
	 * 
	 * @param ObjectInitializer 
	 */
	UE_API ALyraCharacter(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

```
``` cpp

// 胶囊体碰撞预设文件
static FName NAME_LyraCharacterCollisionProfile_Capsule(TEXT("LyraPawnCapsule"));
// 网格体碰撞预设文件
static FName NAME_LyraCharacterCollisionProfile_Mesh(TEXT("LyraPawnMesh"));

ALyraCharacter::ALyraCharacter(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer.SetDefaultSubobjectClass<ULyraCharacterMovementComponent>(ACharacter::CharacterMovementComponentName))
{
	// Avoid ticking characters if possible.
	// 如有可能，请避免勾选相关选项。
	PrimaryActorTick.bCanEverTick = false;
	PrimaryActorTick.bStartWithTickEnabled = false;

	// 设定该角色与客户端视角之间的最大距离的平方值，此值将作为该角色的相关性指标，并将进行复制处理。
	SetNetCullDistanceSquared(900000000.0f);

	UCapsuleComponent* CapsuleComp = GetCapsuleComponent();
	check(CapsuleComp);

	// 设置胶囊的大小，但不会触发渲染或物理更新。当在类构造函数中初始化组件时，这是首选的方法。
	CapsuleComp->InitCapsuleSize(40.0f, 90.0f);
	
	// 设置碰撞配置文件名称
	// 在构造函数中调用此函数时会设置“配置文件名称”属性
	// 这将使当前的“碰撞配置文件名称”变为此处指定的名称，并覆盖原有的碰撞设置	
	CapsuleComp->SetCollisionProfileName(NAME_LyraCharacterCollisionProfile_Capsule);

	
	USkeletalMeshComponent* MeshComp = GetMesh();
	check(MeshComp);
	
	// 将网格旋转至 X 方向朝前，因为其在导出时是以 Y 方向朝前的方式呈现的。
	// 注意这里是 Pitch Yaw Roll ,在蓝图里面呈现的顺序是 Roll Pitch Yaw
	MeshComp->SetRelativeRotation(FRotator(0.0f, -90.0f, 0.0f));  // Rotate mesh to be X forward since it is exported as Y forward.
	// 设置碰撞配置文件名称
	MeshComp->SetCollisionProfileName(NAME_LyraCharacterCollisionProfile_Mesh);

	// 修改移动组件的参数,注意这里使用的是我们自己的移动组件,需要通过ObjectInitializer去指定默认类!
	ULyraCharacterMovementComponent* LyraMoveComp = CastChecked<ULyraCharacterMovementComponent>(GetCharacterMovement());
	// 自定义重力值。对于该角色，重力值会乘以这个数值。
	LyraMoveComp->GravityScale = 1.0f;
	// 最大加速度（速度的变化率）
	LyraMoveComp->MaxAcceleration = 2400.0f;
	// 用于乘以制动时摩擦力实际值的系数。此系数适用于当前所使用的任何摩擦力值，其可能取决于 bUseSeparateBrakingFriction 的设置。
	// 备注:出于历史原因，默认值为 2，而值为 1 时则会得出正确的阻力计算公式。
	LyraMoveComp->BrakingFrictionFactor = 1.0f;
	
	/**
	 *	制动时所应用的摩擦（阻力）系数（在加速值为 0 的情况下，或者当角色超过最大速度时）；实际使用的值是该系数乘以“制动摩擦系数”。
	 *	在制动过程中，此属性允许您控制在地面移动时所施加的摩擦力大小，即施加一个与当前速度成比例的反向力。制动由摩擦（与速度相关的阻力）和恒定的减速组成。
	 *	这是当前值，在所有移动模式中均使用；如果不需要此值，可以对其进行覆盖或在移动模式改变时使用“单独制动摩擦系数”选项。
	 *	备注:只有在“bUseSeparateBrakingFriction”设置为“真”的情况下才使用，否则将使用当前的摩擦力值，例如“GroundFriction”。
	 */
	LyraMoveComp->BrakingFriction = 6.0f;
	
	/**
	 * 影响移动控制的设置。
	 * 数值越大，方向改变的速度就越快。
	 * 如果“使用单独的制动摩擦力”选项为“否”，那么它还会影响在制动时更快地停止的能力（当加速度为零时），此时该值会乘以“制动摩擦力系数”。
	 * 在制动时，此属性允许您控制在地面移动时施加的摩擦力大小，施加一个与当前速度成比例的反向力。
	 * 这可以用来模拟诸如冰面或油面等滑溜溜的表面，通过更改该值（可能基于站立的物体所接触的材质）来实现。
	 */
	LyraMoveComp->GroundFriction = 8.0f;
	
	// 行走时减速且不加速。这是一种持续的反向力，会以固定的速度值直接降低速度。
	LyraMoveComp->BrakingDecelerationWalking = 1400.0f;
	
	/**
	 * 如果情况属实，则根据控制器所期望的旋转方向（通常是 Controller->ControlRotation）平稳地将角色旋转过去，使用 RotationRate 作为旋转变化的速率。
	 * 此设置会覆盖 OrientRotationToMovement。
	 * 通常您需要确保其他设置已清除，例如 Character 上的 bUseControllerRotationYaw。
	 */
	LyraMoveComp->bUseControllerDesiredRotation = false;
	
	/**
	 * 	如果情况属实，则应将角色朝向加速度的方向旋转，使用“旋转速率”作为旋转变化的速率。
	 * 	此设置会覆盖“UseControllerDesiredRotation”选项。通常您需要确保其他设置已清除，例如在角色上设置“bUseControllerRotationYaw”。
	 */
	LyraMoveComp->bOrientRotationToMovement = false;
	
	// 每秒的旋转变化量，当 UseControllerDesiredRotation 或 OrientRotationToMovement 为真时使用。对于无限旋转速率和瞬间旋转，可设置负值。
	LyraMoveComp->RotationRate = FRotator(0.0f, 720.0f, 0.0f);

	// 根节点动画时是否允许物理旋转
	LyraMoveComp->bAllowPhysicsRotationDuringAnimRootMotion = false;
	
	/**
	 * 人工智能导航/路径规划中所使用的“智能体”（或“兵卒”）的表示属性。
	 * 定义组件可如何移动的属性。
	 * 若为真，则此兵种具备蹲伏的能力。
	 * 
	 */
	LyraMoveComp->GetNavAgentPropertiesRef().bCanCrouch = true;
	
	// 如果属实的话，角色在蹲伏状态下是可以从边缘处滑落下去的。
	LyraMoveComp->bCanWalkOffLedgesWhenCrouching = true;

	// 在蹲伏时设置碰撞半高度，并更新相关计算。
	LyraMoveComp->SetCrouchedHalfHeight(65.0f);

	// 创建角色拓展组件
	PawnExtComponent = CreateDefaultSubobject<ULyraPawnExtensionComponent>(TEXT("PawnExtensionComponent"));
	// ASC初始化完成后调用,用于初始化生命值组件绑定属性,和Tag容器接口的初始化
	PawnExtComponent->OnAbilitySystemInitialized_RegisterAndCall(FSimpleMulticastDelegate::FDelegate::CreateUObject(this, &ThisClass::OnAbilitySystemInitialized));
	// ASC移除后调用,用于生命值组件取消绑定属性
	PawnExtComponent->OnAbilitySystemUninitialized_Register(FSimpleMulticastDelegate::FDelegate::CreateUObject(this, &ThisClass::OnAbilitySystemUninitialized));

	// 创建生命值组件
	HealthComponent = CreateDefaultSubobject<ULyraHealthComponent>(TEXT("HealthComponent"));
	// 绑定生命值组件上关于死亡的流程,开始死亡时禁用移动和关闭碰撞
	HealthComponent->OnDeathStarted.AddDynamic(this, &ThisClass::OnDeathStarted);
	// 绑定生命值组件上关于死亡的流程,死亡完成时,出发蓝图调用,并最终销毁
	HealthComponent->OnDeathFinished.AddDynamic(this, &ThisClass::OnDeathFinished);

	// 创建相机组件
	CameraComponent = CreateDefaultSubobject<ULyraCameraComponent>(TEXT("CameraComponent"));
	// 将该组件的位置相对于其父组件进行设定
	CameraComponent->SetRelativeLocation(FVector(-300.0f, 0.0f, 75.0f));
	
	/** 若为真，则当此兵种由玩家控制器控制时，其俯仰角度将与控制器的控制旋转角度保持一致。*/
	bUseControllerRotationPitch = false;

	// 如果情况属实，那么这个兵的偏航角度将会被更新，以与控制器的控制旋转偏航角度保持一致（如果该兵是由玩家控制器控制的）
	bUseControllerRotationYaw = true;

	// 如果情况属实，那么这个“兵”的动作将会被更新，使其与“控制器”的“控制旋转”动作相匹配（如果该“兵”是由“玩家控制器”控制的的话）。
	bUseControllerRotationRoll = false;
	
	// 基底眼睛所在位置距离碰撞中心的垂直高度。
	BaseEyeHeight = 80.0f;

	// 默认的蹲伏时眼睛高度
	CrouchedEyeHeight = 50.0f;
}
```

### 获取游戏玩法类
``` cpp
ALyraPlayerController* ALyraCharacter::GetLyraPlayerController() const
{
	return CastChecked<ALyraPlayerController>(GetController(), ECastCheckedType::NullAllowed);
}

ALyraPlayerState* ALyraCharacter::GetLyraPlayerState() const
{
	return CastChecked<ALyraPlayerState>(GetPlayerState(), ECastCheckedType::NullAllowed);
}

```

### ASC获取
注意这里获取的ASC是通过PlayerState获取的.
PawnExtComponent会在角色初始化过程中在正确的时机持有ASC的指针!
角色初始化流程不在这里讲.
``` cpp
ULyraAbilitySystemComponent* ALyraCharacter::GetLyraAbilitySystemComponent() const
{
	return Cast<ULyraAbilitySystemComponent>(GetAbilitySystemComponent());
}

UAbilitySystemComponent* ALyraCharacter::GetAbilitySystemComponent() const
{
	if (PawnExtComponent == nullptr)
	{
		return nullptr;
	}

	return PawnExtComponent->GetLyraAbilitySystemComponent();
}
```

### Tag的转发获取

``` cpp
	// 走ASC 匹配Tag容器
	UE_API virtual void GetOwnedGameplayTags(FGameplayTagContainer& TagContainer) const override;
	UE_API virtual bool HasMatchingGameplayTag(FGameplayTag TagToCheck) const override;
	UE_API virtual bool HasAllMatchingGameplayTags(const FGameplayTagContainer& TagContainer) const override;
	UE_API virtual bool HasAnyMatchingGameplayTags(const FGameplayTagContainer& TagContainer) const override;

```
```cpp
void ALyraCharacter::GetOwnedGameplayTags(FGameplayTagContainer& TagContainer) const
{
	if (const ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
	{
		LyraASC->GetOwnedGameplayTags(TagContainer);
	}
}

bool ALyraCharacter::HasMatchingGameplayTag(FGameplayTag TagToCheck) const
{
	if (const ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
	{
		return LyraASC->HasMatchingGameplayTag(TagToCheck);
	}

	return false;
}

bool ALyraCharacter::HasAllMatchingGameplayTags(const FGameplayTagContainer& TagContainer) const
{
	if (const ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
	{
		return LyraASC->HasAllMatchingGameplayTags(TagContainer);
	}

	return false;
}

bool ALyraCharacter::HasAnyMatchingGameplayTags(const FGameplayTagContainer& TagContainer) const
{
	if (const ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
	{
		return LyraASC->HasAnyMatchingGameplayTags(TagContainer);
	}

	return false;
}

```

### 重要度管理器
该版本未实现.保留接口.
``` cpp

void ALyraCharacter::BeginPlay()
{
	Super::BeginPlay();

	UWorld* World = GetWorld();

	const bool bRegisterWithSignificanceManager = !IsNetMode(NM_DedicatedServer);
	if (bRegisterWithSignificanceManager)
	{
		if (ULyraSignificanceManager* SignificanceManager = USignificanceManager::Get<ULyraSignificanceManager>(World))
		{
//@TODO: SignificanceManager->RegisterObject(this, (EFortSignificanceType)SignificanceType);
		}
	}
}

void ALyraCharacter::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
	Super::EndPlay(EndPlayReason);

	UWorld* World = GetWorld();

	const bool bRegisterWithSignificanceManager = !IsNetMode(NM_DedicatedServer);
	if (bRegisterWithSignificanceManager)
	{
		if (ULyraSignificanceManager* SignificanceManager = USignificanceManager::Get<ULyraSignificanceManager>(World))
		{
			SignificanceManager->UnregisterObject(this);
		}
	}
}
```
需要进行配置
在DefaultEngine.ini
``` ini
[/Script/SignificanceManager.SignificanceManager]
SignificanceManagerClassName=/Script/LyraGame.LyraSignificanceManager

```
``` cpp
UCLASS()
class ULyraSignificanceManager : public USignificanceManager
{
	GENERATED_BODY()

};

```
### 队伍切换
当控制前变更,角色被占用或者接触占用时都会触发队伍信息的变更.
``` cpp
	//~APawn interface
	// 控制器变更了 所以要重新获取队伍值 并将其改变传递出去
	UE_API virtual void NotifyControllerChanged() override;
	//~End of APawn interface

	// 当此角色被附身时调用。仅在服务器（或独立运行时）调用。
	// 绑定和传递队伍变更信息.
	// 通知角色拓展组件控制器变更了.  尝试推进角色初始化流程
	UE_API virtual void PossessedBy(AController* NewController) override;
	// 当我们的控制器不再控制我们时会触发此事件。此事件仅在服务器端（或在独立运行的情况下）才会触发。
	// 重置队伍信息
	// 通知角色拓展组件控制器变更了.
	UE_API virtual void UnPossessed() override;

```
``` cpp
	//~ILyraTeamAgentInterface interface
	// 修改队伍ID 理论上不能直接修改 需要接收所属控制器的变动 但是需要考虑没有控制器以及非服务器上的错误反馈
	UE_API virtual void SetGenericTeamId(const FGenericTeamId& NewTeamID) override;
	// 获取队伍ID
	UE_API virtual FGenericTeamId GetGenericTeamId() const override;
	// 获取队伍绑定的代理 用于向更下级传递
	UE_API virtual FOnLyraTeamIndexChangedDelegate* GetOnTeamIndexChangedDelegate() override;
	//~End of ILyraTeamAgentInterface interface

```
``` cpp
	// 队伍ID
	UPROPERTY(ReplicatedUsing = OnRep_MyTeamID)
	FGenericTeamId MyTeamID;

	// 向下传递的队伍绑定代理
	UPROPERTY()
	FOnLyraTeamIndexChangedDelegate OnTeamChangedDelegate;

protected:
	// Called to determine what happens to the team ID when possession ends
	// 用于确定当控球权结束时团队标识将如何处理的函数
	virtual FGenericTeamId DetermineNewTeamAfterPossessionEnds(FGenericTeamId OldTeamID) const
	{
		// This could be changed to return, e.g., OldTeamID if you want to keep it assigned afterwards, or return an ID for some neutral faction, or etc...
		// 这可以修改为返回诸如“OldTeamID”之类的值（如果您希望之后仍保留该团队的标识，则可采用此方式），或者返回某个中立阵营的标识，或者等等。
		return FGenericTeamId::NoTeam;
	}

	// 队伍信息的属性同步
	UFUNCTION()
	UE_API void OnRep_MyTeamID(FGenericTeamId OldTeamID);

```
### 加速度同步
#### 传输结构体定义
``` cpp
/**
 *
 * FLyraReplicatedAcceleration: Compressed representation of acceleration
 * 加速度的压缩表示
 * 
 */
USTRUCT()
struct FLyraReplicatedAcceleration
{
	GENERATED_BODY()

	// X-Y 加速度分量的方向，已量化处理以表示范围[0, 2π]
	UPROPERTY()
	uint8 AccelXYRadians = 0;	// Direction of XY accel component, quantized to represent [0, 2*pi]

	// XY 分量的加速度率，已量化处理以表示范围在 [0， 最大加速度] 之间。
	UPROPERTY()
	uint8 AccelXYMagnitude = 0;	//Accel rate of XY component, quantized to represent [0, MaxAcceleration]

	// 原始的加速度率分量，经过量化处理以表示范围为[-最大加速度值，最大加速度值]的数值。
	UPROPERTY()
	int8 AccelZ = 0;	// Raw Z accel rate component, quantized to represent [-MaxAcceleration, MaxAcceleration]
};

```
#### 笛卡尔左边转换为极坐标
``` cpp
void ALyraCharacter::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
	Super::PreReplication(ChangedPropertyTracker);

	if (UCharacterMovementComponent* MovementComponent = GetCharacterMovement())
	{
		// Compress Acceleration: XY components as direction + magnitude, Z component as direct value
		// 压缩加速：XY 分量表示方向和大小，Z 分量表示直接数值
		const double MaxAccel = MovementComponent->MaxAcceleration;
		const FVector CurrentAccel = MovementComponent->GetCurrentAcceleration();
		double AccelXYRadians, AccelXYMagnitude;
		// 将给定的笛卡尔坐标对转换为极坐标系。
		FMath::CartesianToPolar(CurrentAccel.X, CurrentAccel.Y, AccelXYMagnitude, AccelXYRadians);

		ReplicatedAcceleration.AccelXYRadians   = FMath::FloorToInt((AccelXYRadians / TWO_PI) * 255.0);     // [0, 2PI] -> [0, 255]
		ReplicatedAcceleration.AccelXYMagnitude = FMath::FloorToInt((AccelXYMagnitude / MaxAccel) * 255.0);	// [0, MaxAccel] -> [0, 255]
		ReplicatedAcceleration.AccelZ           = FMath::FloorToInt((CurrentAccel.Z / MaxAccel) * 127.0);   // [-MaxAccel, MaxAccel] -> [-127, 127]
	}
}


```
``` cpp
void ALyraCharacter::GetLifetimeReplicatedProps(TArray< FLifetimeProperty >& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);
	
	// 注意这里的同步规则,只同步模拟对象
	DOREPLIFETIME_CONDITION(ThisClass, ReplicatedAcceleration, COND_SimulatedOnly);
	
	DOREPLIFETIME(ThisClass, MyTeamID)
}

```
#### 属性同步到客户端传递给移动组件
``` cpp
void ALyraCharacter::OnRep_ReplicatedAcceleration()
{
	if (ULyraCharacterMovementComponent* LyraMovementComponent = Cast<ULyraCharacterMovementComponent>(GetCharacterMovement()))
	{
		// Decompress Acceleration
		// 解压加速作用
		const double MaxAccel         = LyraMovementComponent->MaxAcceleration;
		const double AccelXYMagnitude = double(ReplicatedAcceleration.AccelXYMagnitude) * MaxAccel / 255.0; // [0, 255] -> [0, MaxAccel]
		const double AccelXYRadians   = double(ReplicatedAcceleration.AccelXYRadians) * TWO_PI / 255.0;     // [0, 255] -> [0, 2PI]

		FVector UnpackedAcceleration(FVector::ZeroVector);
		FMath::PolarToCartesian(AccelXYMagnitude, AccelXYRadians, UnpackedAcceleration.X, UnpackedAcceleration.Y);
		UnpackedAcceleration.Z = double(ReplicatedAcceleration.AccelZ) * MaxAccel / 127.0; // [-127, 127] -> [-MaxAccel, MaxAccel]

		LyraMovementComponent->SetReplicatedAcceleration(UnpackedAcceleration);
	}
}
```
移动组件的内容在其他节讲解!这里知道客户端的移动组件收到了服务器压缩同步过来的加速度就行.
我们并没有开启引擎自带的加速度同步!是手动计算然后属性同步过来的加速度

### 移动数据同步
#### 结构体定义
``` cpp

/** The type we use to send FastShared movement updates. */
/** 我们用于发送快速共享移动更新的类型。*/
USTRUCT()
struct FSharedRepMovement
{
	GENERATED_BODY()

	FSharedRepMovement();

	// 从角色填充RepMovement
	bool FillForCharacter(ACharacter* Character);
	// 判断当前同步数据是否与人物的数据相等
	bool Equals(const FSharedRepMovement& Other, ACharacter* Character) const;

	// 填充数据
	bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess);

	// 我们“根组件”的重复运动数据。该结构用于高效复制，因为速度和位置通常会一起复制（这可以节省一个复制索引），而速度.Z 值通常为零（大多数位置复制是针对行走的兵类角色的）。
	UPROPERTY(Transient)
	FRepMovement RepMovement;

	UPROPERTY(Transient)
	float RepTimeStamp = 0.0f;

	UPROPERTY(Transient)
	uint8 RepMovementMode = 0;

	UPROPERTY(Transient)
	bool bProxyIsJumpForceApplied = false;

	UPROPERTY(Transient)
	bool bIsCrouched = false;
};

// 参考Class.h里面中关于此结构体的描述
// 在FastArray中同步标签时,我们之前使用了WithNetDeltaSerializer -结构体具有一个名为“NetDeltaSerialize”的函数，用于将之前“NetSerialize”操作中状态的差异进行序列化处理。
template<>
struct TStructOpsTypeTraits<FSharedRepMovement> : public TStructOpsTypeTraitsBase2<FSharedRepMovement>
{
	enum
	{
		// 该结构体具有一个“NetSerialize”函数，用于将其状态序列化为用于网络复制的“FArchive”对象。
		WithNetSerializer = true,
		// 结构体有一个名为“NetSerialize”的函数，该函数在进行状态序列化时无需使用包映射。
		WithNetSharedSerialization = true,
	};
};

```

#### LyraReplicationGraph调用UpdateSharedReplication
[Replication Graph](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/replication-graph-in-unreal-engine)

``` cpp
UCLASS(MinimalAPI, Config = Game, Meta = (ShortTooltip = "The base character pawn class used by this project."))
class ALyraCharacter : public AModularCharacter, public IAbilitySystemInterface, public IGameplayCueInterface, public IGameplayTagAssetInterface, public ILyraTeamAgentInterface
{
	GENERATED_BODY()

	/** RPCs that is called on frames when default property replication is skipped. This replicates a single movement update to everyone. */
	/** 当默认属性复制被跳过时，在帧上调用的远程过程调用。此过程会将单次移动更新复制给所有用户。*/
	UFUNCTION(NetMulticast, unreliable)
	UE_API void FastSharedReplication(const FSharedRepMovement& SharedRepMovement);

	// Last FSharedRepMovement we sent, to avoid sending repeatedly.
	// 我们上一次发送的共享移动数据，目的是避免重复发送。
	FSharedRepMovement LastSharedReplication;

	// 由LyraReplicationGraph调用,更新同步的移动信息
	UE_API virtual bool UpdateSharedReplication();

}
```

``` cpp
void ULyraReplicationGraph::InitGlobalActorClassSettings()
{
	// ------------------------------------------------------------------------------------------------------
	//	Setup FastShared replication for pawns. This is called up to once per frame per pawn to see if it wants
	//	to send a FastShared update to all relevant connections.
	// ------------------------------------------------------------------------------------------------------
	CharacterClassRepInfo.FastSharedReplicationFunc = [](AActor* Actor)
	{
		bool bSuccess = false;
		if (ALyraCharacter* Character = Cast<ALyraCharacter>(Actor))
		{
			bSuccess = Character->UpdateSharedReplication();
		}
		return bSuccess;
	};

}
```

#### 服务端计算数据
``` cpp

bool ALyraCharacter::UpdateSharedReplication()
{
	if (GetLocalRole() == ROLE_Authority)
	{
		FSharedRepMovement SharedMovement;
		if (SharedMovement.FillForCharacter(this))
		{
			// Only call FastSharedReplication if data has changed since the last frame.
			// Skipping this call will cause replication to reuse the same bunch that we previously
			// produced, but not send it to clients that already received. (But a new client who has not received
			// it, will get it this frame)
			
			// 仅在数据自上一帧以来发生更改时才调用 FastSharedReplication 函数。
			// 忽略此调用将导致复制操作重复使用我们之前生成的相同数据组，但不会将其发送给已经接收过该数据的客户端。（但对于尚未接收该数据的新客户端，它将在本帧中获得该数据）
			if (!SharedMovement.Equals(LastSharedReplication, this))
			{
				LastSharedReplication = SharedMovement;
				/**
				 * 将 CharacterMovement 的 MovementMode（以及自定义模式）设置为可进行网络复制的值，以便在模拟代理中使用。
				 * 使用 CharacterMovementComponent:：UnpackNetworkMovementMode() 函数来将其转换为可读形式。
				 * 此函数会设置可压缩的 ReplicatedMovementMode 值。
				 * 
				 */
				SetReplicatedMovementMode(SharedMovement.RepMovementMode);

				FastSharedReplication(SharedMovement);
			}
			return true;
		}
	}

	// We cannot fastrep right now. Don't send anything.
	// 目前我们无法快速处理。请不要发送任何信息。
	return false;
}


```
``` cpp
bool FSharedRepMovement::FillForCharacter(ACharacter* Character)
{
	if (USceneComponent* PawnRootComponent = Character->GetRootComponent())
	{
		UCharacterMovementComponent* CharacterMovement = Character->GetCharacterMovement();
		
		// 根据角色所在的“世界”位置，将本地起始位置重新定位至零世界原点值。
		// 在世界空间中的位置
		RepMovement.Location = FRepMovement::RebaseOntoZeroOrigin(PawnRootComponent->GetComponentLocation(), Character);
		/** 返回组件在世界空间中的旋转角度 */
		// 当前的旋转状态
		RepMovement.Rotation = PawnRootComponent->GetComponentRotation();
		/** 当前更新组件的速度。*/
		// 世界空间中组件的速度
		RepMovement.LinearVelocity = CharacterMovement->Velocity;
		// 移动模式打包 主要是用于节约带宽传输
		RepMovementMode = CharacterMovement->PackNetworkMovementMode();
		/** GetProxyIsJumpForceApplied:表示此角色当前正受到跳跃力的作用（若 JumpMaxHoldTime 不为零）。IsJumpProvidingForce() 函数也会处理此情况。此函数返回 bProxyIsJumpForceApplied 属性 */
		/** JumpForceTimeRemaining:当 JumpMaxHoldTime 不为零时，剩余的跳跃力量持续时间。*/
		bProxyIsJumpForceApplied = Character->GetProxyIsJumpForceApplied() || (Character->JumpForceTimeRemaining > 0.0f);
		/** 由角色移动操作设定，用于指示当前该角色处于蹲伏状态。此函数返回 bIsCrouched 值。*/
		bIsCrouched = Character->IsCrouched();

		// Timestamp is sent as zero if unused
		// 若未使用，则时间戳将被发送为零值
		/**
		 * bNetworkAlwaysReplicateTransformUpdateTimestamp
		 * 服务器上使用的标志，用于决定是否始终将“ReplicatedServerLastTransformUpdateTimeStamp”数据复制给客户端。
		 * 通常只有在角色移动的网络平滑模式设置为线性平滑（在服务器端）时才会发送此数据，以节省带宽。
		 * 将此标志设置为“真”将强制该时间戳进行复制，无论服务器是否知晓平滑模式，或者如果该时间戳用于其他目的的话。
		 *
		 */
		// NetworkSmoothingMode:网络游戏中模拟代理对象的平滑处理模式。
		// Linear:从源点到目标点进行线性插值。
		if ((CharacterMovement->NetworkSmoothingMode == ENetworkSmoothingMode::Linear) || CharacterMovement->bNetworkAlwaysReplicateTransformUpdateTimestamp)
		{
			RepTimeStamp = CharacterMovement->GetServerLastTransformUpdateTimeStamp();
		}
		else
		{
			RepTimeStamp = 0.f;
		}

		return true;
	}
	return false;
}

```
#### 客户端更新数据
``` cpp

void ALyraCharacter::FastSharedReplication_Implementation(const FSharedRepMovement& SharedRepMovement)
{
	/** 如果我们当前正在播放回放，则返回 true */
	if (GetWorld()->IsPlayingReplay())
	{
		return;
	}

	// Timestamp is checked to reject old moves.
	// 会检查时间戳以剔除过时的交易记录。
	if (GetLocalRole() == ROLE_SimulatedProxy)
	{
		// Timestamp
		// 时间戳
		/** 用于记录角色移动的“服务器上最新变换更新时间戳”的值，该值会被复制到模拟代理中。*/
		SetReplicatedServerLastTransformUpdateTimeStamp(SharedRepMovement.RepTimeStamp);

		// Movement mode
		if (GetReplicatedMovementMode() != SharedRepMovement.RepMovementMode)
		{
			SetReplicatedMovementMode(SharedRepMovement.RepMovementMode);
			/** 当网络化移动模式已完成复制时为真。*/
			GetCharacterMovement()->bNetworkMovementModeChanged = true;
			// 当接收到模拟代理的网络复制更新时为真。
			GetCharacterMovement()->bNetworkUpdateReceived = true;
		}

		// Location, Rotation, Velocity, etc.
		// 位置、旋转、速度等
		/**
		 * GetReplicatedMovement_Mutable
		 * 获取对“ReplicatedMovement”对象的引用，并期望该对象会被修改。*
		 * 这样做是为了避免子类需要直接访问“复制运动”属性
		 * 因此，之后可以将其设为私有属性。
		 * 
		 */
		FRepMovement& MutableRepMovement = GetReplicatedMovement_Mutable();
		MutableRepMovement = SharedRepMovement.RepMovement;

		// This also sets LastRepMovement
		// 这样也设置了“最后移动状态”
		OnRep_ReplicatedMovement();

		// Jump force
		// 跳跃力
		// 表示该角色当前正受到跳跃力的作用（若 JumpMaxHoldTime 不为零）。IsJumpProvidingForce() 函数也会处理此情况。此函数会设置 bProxyIsJumpForceApplied
		SetProxyIsJumpForceApplied(SharedRepMovement.bProxyIsJumpForceApplied);

		// Crouch
		// 蹲下
		if (IsCrouched() != SharedRepMovement.bIsCrouched)
		{
			SetIsCrouched(SharedRepMovement.bIsCrouched);
			OnRep_IsCrouched();
		}
	}
}

```
### 业务转发给生命值组件

#### 绑定属性
此处不详细展开.因为涉及到ASC的属性绑定.生命值转发到蓝图调用血条等等.
``` cpp
	// 由角色拓展组件的代理调用,ASC初始化后触发,更新生命值组件和Tag接口
	UE_API virtual void OnAbilitySystemInitialized();
	// 由角色拓展组件的代理调用,ASC移除后触发,取消生命值组件的绑定属性
	UE_API virtual void OnAbilitySystemUninitialized();

```
``` cpp
void ALyraCharacter::OnAbilitySystemInitialized()
{
	ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent();
	check(LyraASC);

	HealthComponent->InitializeWithAbilitySystem(LyraASC);

	InitializeGameplayTags();
}
void ALyraCharacter::OnAbilitySystemUninitialized()
{
	HealthComponent->UninitializeFromAbilitySystem();
}
```
#### 死亡流程
此处不详细展开.
``` cpp
	// 创建生命值组件
	HealthComponent = CreateDefaultSubobject<ULyraHealthComponent>(TEXT("HealthComponent"));
	// 绑定生命值组件上关于死亡的流程,开始死亡时禁用移动和关闭碰撞
	HealthComponent->OnDeathStarted.AddDynamic(this, &ThisClass::OnDeathStarted);
	// 绑定生命值组件上关于死亡的流程,死亡完成时,出发蓝图调用,并最终销毁
	HealthComponent->OnDeathFinished.AddDynamic(this, &ThisClass::OnDeathFinished);


```
``` cpp
void ALyraCharacter::OnDeathStarted(AActor*)
{
	DisableMovementAndCollision();
}

void ALyraCharacter::OnDeathFinished(AActor*)
{
	GetWorld()->GetTimerManager().SetTimerForNextTick(this, &ThisClass::DestroyDueToDeath);
}


```
``` cpp
void ALyraCharacter::OnDeathStarted(AActor*)
{
	DisableMovementAndCollision();
}

void ALyraCharacter::OnDeathFinished(AActor*)
{
	GetWorld()->GetTimerManager().SetTimerForNextTick(this, &ThisClass::DestroyDueToDeath);
}


void ALyraCharacter::DisableMovementAndCollision()
{
	if (GetController())
	{
		GetController()->SetIgnoreMoveInput(true);
	}

	UCapsuleComponent* CapsuleComp = GetCapsuleComponent();
	check(CapsuleComp);
	CapsuleComp->SetCollisionEnabled(ECollisionEnabled::NoCollision);
	CapsuleComp->SetCollisionResponseToAllChannels(ECR_Ignore);

	ULyraCharacterMovementComponent* LyraMoveComp = CastChecked<ULyraCharacterMovementComponent>(GetCharacterMovement());
	LyraMoveComp->StopMovementImmediately();
	LyraMoveComp->DisableMovement();
}

void ALyraCharacter::DestroyDueToDeath()
{
	K2_OnDeathFinished();

	UninitAndDestroy();
}


void ALyraCharacter::UninitAndDestroy()
{
	if (GetLocalRole() == ROLE_Authority)
	{
		/** 调用此函数可安全地将兵卒从其控制器处分离出来，要知道我们很快就会被摧毁。*/
		DetachFromControllerPendingDestroy();
		SetLifeSpan(0.1f);
	}

	// Uninitialize the ASC if we're still the avatar actor (otherwise another pawn already did it when they became the avatar actor)
	// 如果我们仍是角色扮演者，则解除 ASC 的初始化（否则，当其他玩家成为角色扮演者时，他们已经完成了这一操作）
	if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
	{
		if (LyraASC->GetAvatarActor() == this)
		{
			PawnExtComponent->UninitializeAbilitySystem();
		}
	}
	// 将该角色设置为游戏中的隐藏状态 true 关联组件也隐藏
	SetActorHiddenInGame(true);
}


```

#### 掉出边界
``` cpp
	/** 当角色“安全地”脱离游戏世界（低于“杀戮者”等角色的位置时）时会调用此函数 */
	// 掉出场景限制高度时 会调用这个函数
	// 通过生命值组件 造成一次足以自我销毁的伤害即可
	UE_API virtual void FellOutOfWorld(const class UDamageType& dmgType) override;

```
``` cpp
void ALyraCharacter::FellOutOfWorld(const class UDamageType& dmgType)
{
	HealthComponent->DamageSelfDestruct(/*bFellOutOfWorld=*/ true);
}

```
### 业务转发给PawnExt
``` cpp
void ALyraCharacter::OnRep_Controller()
{
	Super::OnRep_Controller();

	PawnExtComponent->HandleControllerChanged();
}

void ALyraCharacter::OnRep_PlayerState()
{
	Super::OnRep_PlayerState();

	PawnExtComponent->HandlePlayerStateReplicated();
}

void ALyraCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);

	PawnExtComponent->SetupPlayerInputComponent();
}


```


### 移动tag的处理
方便从引擎底层的移动枚举转换成我们识别的移动tag

#### 移动枚举
``` cpp

/** Movement modes for Characters. */
UENUM(BlueprintType)
enum EMovementMode : int
{
	/** None (movement is disabled). */
	MOVE_None		UMETA(DisplayName="None"),

	/** Walking on a surface. */
	MOVE_Walking	UMETA(DisplayName="Walking"),

	/** 
	 * Simplified walking on navigation data (e.g. navmesh). 
	 * If GetGenerateOverlapEvents() is true, then we will perform sweeps with each navmesh move.
	 * If GetGenerateOverlapEvents() is false then movement is cheaper but characters can overlap other objects without some extra process to repel/resolve their collisions.
	 */
	MOVE_NavWalking	UMETA(DisplayName="Navmesh Walking"),

	/** Falling under the effects of gravity, such as after jumping or walking off the edge of a surface. */
	MOVE_Falling	UMETA(DisplayName="Falling"),

	/** Swimming through a fluid volume, under the effects of gravity and buoyancy. */
	MOVE_Swimming	UMETA(DisplayName="Swimming"),

	/** Flying, ignoring the effects of gravity. Affected by the current physics volume's fluid friction. */
	MOVE_Flying		UMETA(DisplayName="Flying"),

	/** User-defined custom movement mode, including many possible sub-modes. */
	MOVE_Custom		UMETA(DisplayName="Custom"),

	MOVE_MAX		UMETA(Hidden),
};


```
#### 移动Tag
``` cpp
namespace LyraGameplayTags
{
	// Unreal Movement Modes
	const TMap<uint8, FGameplayTag> MovementModeTagMap =
	{
		{ MOVE_Walking, Movement_Mode_Walking },
		{ MOVE_NavWalking, Movement_Mode_NavWalking },
		{ MOVE_Falling, Movement_Mode_Falling },
		{ MOVE_Swimming, Movement_Mode_Swimming },
		{ MOVE_Flying, Movement_Mode_Flying },
		{ MOVE_Custom, Movement_Mode_Custom }
	};

	// Custom Movement Modes
	const TMap<uint8, FGameplayTag> CustomMovementModeTagMap =
	{
		// Fill these in with your custom modes
	};
}
```
#### 绑定转换
``` cpp
	/**
	 * 由“角色移动组件”调用，以通知角色移动模式已发生改变。
	 * @参数  原有移动模式：改变前的移动模式
	 * @参数  原有自定义模式：若原有移动模式为“自定义”则为自定义模式（仅适用于该情况）
 	 * 
	 */
	UE_API virtual void OnMovementModeChanged(EMovementMode PrevMovementMode, uint8 PreviousCustomMode) override;
	// 用于从移动组件中读取当前的移动类型,然后转换成Tag赋予给ASC.
	UE_API void SetMovementModeTag(EMovementMode MovementMode, uint8 CustomMovementMode, bool bTagEnabled);

		// 初始化游戏玩法Tag 主要用于移动模式从引擎底层的枚举转换到Tag
	UE_API void InitializeGameplayTags();

```
``` cpp
void ALyraCharacter::OnMovementModeChanged(EMovementMode PrevMovementMode, uint8 PreviousCustomMode)
{
	Super::OnMovementModeChanged(PrevMovementMode, PreviousCustomMode);

	ULyraCharacterMovementComponent* LyraMoveComp = CastChecked<ULyraCharacterMovementComponent>(GetCharacterMovement());

	SetMovementModeTag(PrevMovementMode, PreviousCustomMode, false);
	SetMovementModeTag(LyraMoveComp->MovementMode, LyraMoveComp->CustomMovementMode, true);
}

void ALyraCharacter::SetMovementModeTag(EMovementMode MovementMode, uint8 CustomMovementMode, bool bTagEnabled)
{
	if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
	{
		const FGameplayTag* MovementModeTag = nullptr;
		if (MovementMode == MOVE_Custom)
		{
			MovementModeTag = LyraGameplayTags::CustomMovementModeTagMap.Find(CustomMovementMode);
		}
		else
		{
			MovementModeTag = LyraGameplayTags::MovementModeTagMap.Find(MovementMode);
		}

		if (MovementModeTag && MovementModeTag->IsValid())
		{
			LyraASC->SetLooseGameplayTagCount(*MovementModeTag, (bTagEnabled ? 1 : 0));
		}
	}
}


```
``` cpp

void ALyraCharacter::InitializeGameplayTags()
{
	// Clear tags that may be lingering on the ability system from the previous pawn.
	// 清除之前角色在能力系统中可能遗留的无效标签。
	if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
	{
		for (const TPair<uint8, FGameplayTag>& TagMapping : LyraGameplayTags::MovementModeTagMap)
		{
			if (TagMapping.Value.IsValid())
			{
				LyraASC->SetLooseGameplayTagCount(TagMapping.Value, 0);
			}
		}

		for (const TPair<uint8, FGameplayTag>& TagMapping : LyraGameplayTags::CustomMovementModeTagMap)
		{
			if (TagMapping.Value.IsValid())
			{
				LyraASC->SetLooseGameplayTagCount(TagMapping.Value, 0);
			}
		}

		ULyraCharacterMovementComponent* LyraMoveComp = CastChecked<ULyraCharacterMovementComponent>(GetCharacterMovement());
		SetMovementModeTag(LyraMoveComp->MovementMode, LyraMoveComp->CustomMovementMode, true);
	}
}
```
### 蹲伏和跳跃
``` cpp

void ALyraCharacter::ToggleCrouch()
{
	const ULyraCharacterMovementComponent* LyraMoveComp = CastChecked<ULyraCharacterMovementComponent>(GetCharacterMovement());

	if (IsCrouched() || LyraMoveComp->bWantsToCrouch)
	{
		UnCrouch();
	}
	else if (LyraMoveComp->IsMovingOnGround())
	{
		Crouch();
	}
}

void ALyraCharacter::OnStartCrouch(float HalfHeightAdjust, float ScaledHalfHeightAdjust)
{
	if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
	{
		LyraASC->SetLooseGameplayTagCount(LyraGameplayTags::Status_Crouching, 1);
	}


	Super::OnStartCrouch(HalfHeightAdjust, ScaledHalfHeightAdjust);
}

void ALyraCharacter::OnEndCrouch(float HalfHeightAdjust, float ScaledHalfHeightAdjust)
{
	if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
	{
		LyraASC->SetLooseGameplayTagCount(LyraGameplayTags::Status_Crouching, 0);
	}

	Super::OnEndCrouch(HalfHeightAdjust, ScaledHalfHeightAdjust);
}

bool ALyraCharacter::CanJumpInternal_Implementation() const
{
	// same as ACharacter's implementation but without the crouch check
	// 与 ACharacter 的实现方式相同，但不包含蹲伏检查的部分
	return JumpIsAllowedInternal();
}

```



### 代码
``` cpp

/**
 * ALyraCharacter
 *
 *	The base character pawn class used by this project.
 *	Responsible for sending events to pawn components.
 *	New behavior should be added via pawn components when possible.
 *
 *  该项目所使用的基础角色“兵”类。负责向兵类组件发送事件。在可能的情况下，应通过兵类组件来添加新的行为。
 *
 *	
 */
UCLASS(MinimalAPI, Config = Game, Meta = (ShortTooltip = "The base character pawn class used by this project."))
class ALyraCharacter : public AModularCharacter, public IAbilitySystemInterface, public IGameplayCueInterface, public IGameplayTagAssetInterface, public ILyraTeamAgentInterface
{
	GENERATED_BODY()

public:
	/**
	 * 构造函数 非常重要
	 * 修改胶囊体组件的默认参数和碰撞预设
	 * 修改网格体组件的默认参数和碰撞预设
	 * 修改移动组件的默认类和默认参数
	 *
	 * 创建角色拓展组件,绑定ASC初始化和移除时的代理,用于出发生命值组件去绑定属性,用于Tag的接口初始化
	 * 创建生命值组件,用于接收ASC属性的绑定,并接收关于死亡流程的触发和完成
	 * 创建相机组件,设置相对位置
	 *
	 * 修改角色的默认参数
	 * 
	 * @param ObjectInitializer 
	 */
	UE_API ALyraCharacter(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	// 获取玩家控制器,可以为空,不崩溃
	UFUNCTION(BlueprintCallable, Category = "Lyra|Character")
	UE_API ALyraPlayerController* GetLyraPlayerController() const;

	// 获取玩家状态类,可以为空,不崩溃
	UFUNCTION(BlueprintCallable, Category = "Lyra|Character")
	UE_API ALyraPlayerState* GetLyraPlayerState() const;


	// 获取Lyra的ASC组件
	UFUNCTION(BlueprintCallable, Category = "Lyra|Character")
	UE_API ULyraAbilitySystemComponent* GetLyraAbilitySystemComponent() const;
	// 通过角色拓展组件获取ASC组件
	// 注意ASC的组件可能有多个来源,本项目中关于人物主要来自于PlayerState,无论是AI还是玩家
	// 为了组件的初始化时序方便管理,所以都是从角色拓展组件获取
	UE_API virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

	// 走ASC 匹配Tag容器
	UE_API virtual void GetOwnedGameplayTags(FGameplayTagContainer& TagContainer) const override;
	UE_API virtual bool HasMatchingGameplayTag(FGameplayTag TagToCheck) const override;
	UE_API virtual bool HasAllMatchingGameplayTags(const FGameplayTagContainer& TagContainer) const override;
	UE_API virtual bool HasAnyMatchingGameplayTags(const FGameplayTagContainer& TagContainer) const override;

	// 切换蹲伏 由英雄组件接收输入系统进行出发
	UE_API void ToggleCrouch();

	//~AActor interface
	// 无作用
	UE_API virtual void PreInitializeComponents() override;
	
	// 无作用 这里添加了重要度管理器的注册 未实现
	// 需要在DefaultEngine.ini中配置
	// [/Script/SignificanceManager.SignificanceManager]
	// SignificanceManagerClassName=/Script/LyraGame.LyraSignificanceManager
	UE_API virtual void BeginPlay() override;
	// 无作用 这里移除了重要度管理器的注册
	UE_API virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
	/*
	 * 将角色重置至初始状态 - 用于在不重新加载的情况下重新开始关卡时使用。
	 * 关闭移动和碰撞进行销毁.
	 */
	UE_API virtual void Reset() override;
	// 设置网络同步属性 加速度和队伍ID
	UE_API virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
	// 在复制操作即将发生时调用该函数。仅在服务器端调用，若正在录制客户端回放，则还会针对自主代理进行调用。
	// 用于计算加速度 然后同步这个属性
	UE_API virtual void PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker) override;
	//~End of AActor interface

	//~APawn interface
	// 控制器变更了 所以要重新获取队伍值 并将其改变传递出去
	UE_API virtual void NotifyControllerChanged() override;
	//~End of APawn interface

	//~ILyraTeamAgentInterface interface
	// 修改队伍ID 理论上不能直接修改 需要接收所属控制器的变动 但是需要考虑没有控制器以及非服务器上的错误反馈
	UE_API virtual void SetGenericTeamId(const FGenericTeamId& NewTeamID) override;
	// 获取队伍ID
	UE_API virtual FGenericTeamId GetGenericTeamId() const override;
	// 获取队伍绑定的代理 用于向更下级传递
	UE_API virtual FOnLyraTeamIndexChangedDelegate* GetOnTeamIndexChangedDelegate() override;
	//~End of ILyraTeamAgentInterface interface

	/** RPCs that is called on frames when default property replication is skipped. This replicates a single movement update to everyone. */
	/** 当默认属性复制被跳过时，在帧上调用的远程过程调用。此过程会将单次移动更新复制给所有用户。*/
	UFUNCTION(NetMulticast, unreliable)
	UE_API void FastSharedReplication(const FSharedRepMovement& SharedRepMovement);

	// Last FSharedRepMovement we sent, to avoid sending repeatedly.
	// 我们上一次发送的共享移动数据，目的是避免重复发送。
	FSharedRepMovement LastSharedReplication;

	// 由LyraReplicationGraph调用,更新同步的移动信息
	UE_API virtual bool UpdateSharedReplication();

protected:

	// 由角色拓展组件的代理调用,ASC初始化后触发,更新生命值组件和Tag接口
	UE_API virtual void OnAbilitySystemInitialized();
	// 由角色拓展组件的代理调用,ASC移除后触发,取消生命值组件的绑定属性
	UE_API virtual void OnAbilitySystemUninitialized();

	// 当此角色被附身时调用。仅在服务器（或独立运行时）调用。
	// 绑定和传递队伍变更信息.
	// 通知角色拓展组件控制器变更了.  尝试推进角色初始化流程
	UE_API virtual void PossessedBy(AController* NewController) override;
	// 当我们的控制器不再控制我们时会触发此事件。此事件仅在服务器端（或在独立运行的情况下）才会触发。
	// 重置队伍信息
	// 通知角色拓展组件控制器变更了.
	UE_API virtual void UnPossessed() override;

	// 通知角色拓展组件 控制器变更了 尝试推进角色初始化流程
	UE_API virtual void OnRep_Controller() override;
	// 通知角色拓展组件 玩家状态类同步过来了  尝试推进角色初始化流程
	UE_API virtual void OnRep_PlayerState() override;
	
	/** 允许兵种设置自定义输入绑定。当兵种被玩家控制器占有时（通过调用 CreatePlayerInputComponent() 创建的输入组件实现），会触发此函数。  */
	// 通知角色拓展组件 输入组件设置好了 尝试推进角色初始化流程
	UE_API virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;

	// 初始化游戏玩法Tag 主要用于移动模式从引擎底层的枚举转换到Tag
	UE_API void InitializeGameplayTags();

	/** 当角色“安全地”脱离游戏世界（低于“杀戮者”等角色的位置时）时会调用此函数 */
	// 掉出场景限制高度时 会调用这个函数
	// 通过生命值组件 造成一次足以自我销毁的伤害即可
	UE_API virtual void FellOutOfWorld(const class UDamageType& dmgType) override;

	// Begins the death sequence for the character (disables collision, disables movement, etc...)
	// 开始该角色的死亡流程（会禁用碰撞、禁用移动等功能......）
	// 由生命值组件调用
	UFUNCTION()
	UE_API virtual void OnDeathStarted(AActor* OwningActor);

	// Ends the death sequence for the character (detaches controller, destroys pawn, etc...)
	// 结束角色的死亡过程（解除控制器控制、销毁角色等操作）
	// 由生命值组件调用
	UFUNCTION()
	UE_API virtual void OnDeathFinished(AActor* OwningActor);

	// 关闭移动和碰撞
	UE_API void DisableMovementAndCollision();
	// 触发摧毁逻辑
	UE_API void DestroyDueToDeath();
	// 通过在服务端设置生命周期 执行销毁逻辑
	UE_API void UninitAndDestroy();

	// Called when the death sequence for the character has completed
	// 当角色的死亡流程完成时触发此事件
	UFUNCTION(BlueprintImplementableEvent, meta=(DisplayName="OnDeathFinished"))
	UE_API void K2_OnDeathFinished();
	/**
	 * 由“角色移动组件”调用，以通知角色移动模式已发生改变。
	 * @参数  原有移动模式：改变前的移动模式
	 * @参数  原有自定义模式：若原有移动模式为“自定义”则为自定义模式（仅适用于该情况）
 	 * 
	 */
	UE_API virtual void OnMovementModeChanged(EMovementMode PrevMovementMode, uint8 PreviousCustomMode) override;
	// 用于从移动组件中读取当前的移动类型,然后转换成Tag赋予给ASC.
	UE_API void SetMovementModeTag(EMovementMode MovementMode, uint8 CustomMovementMode, bool bTagEnabled);

	// 当角色蹲下时触发此事件。对于非所属角色，通过“bIsCrouched”属性进行复制来触发此事件。
	// 添加和移除蹲伏的Tag
	UE_API virtual void OnStartCrouch(float HalfHeightAdjust, float ScaledHalfHeightAdjust) override;
	// 当角色停止蹲伏时触发。通过“bIsCrouched”属性在非所属角色中触发此事件。
	// 添加和移除蹲伏的Tag
	UE_API virtual void OnEndCrouch(float HalfHeightAdjust, float ScaledHalfHeightAdjust) override;

	// 这里重写了父类逻辑 移除了蹲伏检查的逻辑 
	UE_API virtual bool CanJumpInternal_Implementation() const;

private:

	// 拓展组件 极其重要!
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Lyra|Character", Meta = (AllowPrivateAccess = "true"))
	TObjectPtr<ULyraPawnExtensionComponent> PawnExtComponent;

	// 生命值组件
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Lyra|Character", Meta = (AllowPrivateAccess = "true"))
	TObjectPtr<ULyraHealthComponent> HealthComponent;

	// 相机组件
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Lyra|Character", Meta = (AllowPrivateAccess = "true"))
	TObjectPtr<ULyraCameraComponent> CameraComponent;

	// 用于同步的加速度 在服务端进行压缩 在客户端进行解压 然后 传递给移动组件
	UPROPERTY(Transient, ReplicatedUsing = OnRep_ReplicatedAcceleration)
	FLyraReplicatedAcceleration ReplicatedAcceleration;

	// 队伍ID
	UPROPERTY(ReplicatedUsing = OnRep_MyTeamID)
	FGenericTeamId MyTeamID;

	// 向下传递的队伍绑定代理
	UPROPERTY()
	FOnLyraTeamIndexChangedDelegate OnTeamChangedDelegate;

protected:
	// Called to determine what happens to the team ID when possession ends
	// 用于确定当控球权结束时团队标识将如何处理的函数
	virtual FGenericTeamId DetermineNewTeamAfterPossessionEnds(FGenericTeamId OldTeamID) const
	{
		// This could be changed to return, e.g., OldTeamID if you want to keep it assigned afterwards, or return an ID for some neutral faction, or etc...
		// 这可以修改为返回诸如“OldTeamID”之类的值（如果您希望之后仍保留该团队的标识，则可采用此方式），或者返回某个中立阵营的标识，或者等等。
		return FGenericTeamId::NoTeam;
	}

private:
	// 队伍信息变更时触发
	UFUNCTION()
	UE_API void OnControllerChangedTeam(UObject* TeamAgent, int32 OldTeam, int32 NewTeam);

	// 接收成员变量加速度的属性同步,从极坐标转换为笛卡尔坐标
	UFUNCTION()
	UE_API void OnRep_ReplicatedAcceleration();

	// 队伍信息的属性同步
	UFUNCTION()
	UE_API void OnRep_MyTeamID(FGenericTeamId OldTeamID);
};
```
## 总结
3C中的Character极其极其极其重要!但是在Lyra项目中已经尽可能简化了.很多功能都分化到了不同组件上,并且会有一个PawnExtension组件来进行统一管理.后续讲解.