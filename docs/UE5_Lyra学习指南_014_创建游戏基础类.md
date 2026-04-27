# UE5_Lyra学习指南_014_创建游戏基础类 

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)

文章内容由小刚撰写,采用了以下多种方式:

1.口述转文字
2.AI重构
3.参考引擎源码
4.Lyra工程源码
5.结合社区论坛各位大佬的解析

- [UE5\_Lyra学习指南\_014\_创建游戏基础类](#ue5_lyra学习指南_014_创建游戏基础类)
	- [概述](#概述)
	- [1.创建LyraGameMode](#1创建lyragamemode)
	- [2.创建LyraGameState](#2创建lyragamestate)
	- [3.创建ULyraLocalPlayer](#3创建ulyralocalplayer)
	- [4.创建LyraPlayerController](#4创建lyraplayercontroller)
	- [5.创建ALyraReplayPlayerController](#5创建alyrareplayplayercontroller)
	- [6.创建ALyraPlayerState](#6创建alyraplayerstate)
	- [7.创建LyraCharacter](#7创建lyracharacter)
	- [8.创建LyraHUD](#8创建lyrahud)
	- [9.创建LyraGameSession](#9创建lyragamesession)
	- [10.关联到GameMode](#10关联到gamemode)
	- [11.创建LyraGameViewportClient](#11创建lyragameviewportclient)
	- [12.创建LyraSettingsLocal](#12创建lyrasettingslocal)
	- [总结](#总结)



## 概述

本节比较轻松,主要是创建我们项目所使用的一些游戏基础类.

## 1.创建LyraGameMode
创建我们项目自定义的GameMode.注意继承关系!ModeBase要和StateBase一起配套使用,不能混用!!!
注意,此处我们继承的AModularGameModeBase
``` cpp
/**
 * ALyraGameMode
 *
 *	The base game mode class used by this project.
 *	该项目所使用的基础游戏模式类。
 */
UCLASS(MinimalAPI, Config = Game, Meta = (ShortTooltip = "The base game mode class used by this project."))
class ALyraGameMode : public AModularGameModeBase
{
	GENERATED_BODY()

public:
	//构造函数 用于指定项目使用的默认类
	UE_API ALyraGameMode(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

};
```
这是来自引擎源码的关于GameMode的注释.
``` txt
“GameModeBase”定义了正在进行的游戏内容。它控制着游戏规则、得分机制、允许在该游戏类型中存在的角色类型，以及谁可以进入游戏。
它仅在服务器端实例化，永远不会在客户端存在。
当在 C++ 的 UGameEngine::LoadMap() 中为游戏玩法初始化关卡时，会实例化“GameModeBase”角色。
此“GameMode”角色的类是由（按顺序）以下内容决定的：URL 中的“？game=xxx”、在世界设置中设置的“GameMode”覆盖值，或者游戏项目设置中设置的“DefaultGameMode”条目。
```
## 2.创建LyraGameState
我们创建的GameState是继承于AModularGameStateBase,注意Base!

``` cpp
/**
 * ALyraGameState
 *
 *	The base game state class used by this project.
 * 本项目所使用的基础游戏状态类。
 */
UCLASS(MinimalAPI, Config = Game)
class ALyraGameState : public AModularGameStateBase
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
	
};
```
注意在ModularGameStateBase里面,我们注册为了接受者,在游戏结束时也需要取消注册.
有时候这个游戏对象真正介入游戏过程,其实是在BeginPlay之后.
``` cpp
void AModularGameStateBase::PreInitializeComponents()
{
	Super::PreInitializeComponents();
	
	/** Adds an actor as a receiver for components (automatically finding the manager for the actor's  game instance). If it passes the actorclass filter on requests it will get the components. */
	/** 将一个角色添加为组件的接收者（会自动查找该角色所属游戏实例的管理器）。如果该角色符合请求中的角色类过滤条件，则会获取相关组件。*/
	UGameFrameworkComponentManager::AddGameFrameworkComponentReceiver(this);
}

void AModularGameStateBase::BeginPlay()
{
	/** Sends an arbitrary extension event that can be listened for by other systems */
	/** 发送一个任意的扩展事件，其他系统可以监听该事件 */
	UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(this, UGameFrameworkComponentManager::NAME_GameActorReady);

	Super::BeginPlay();
}

void AModularGameStateBase::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
	/** Removes an actor as a receiver for components (automatically finding the manager for the actor's game instance). */
	/** 移除一个角色作为组件接收者的状态（会自动查找该角色所属游戏实例的管理器）。*/
	UGameFrameworkComponentManager::RemoveGameFrameworkComponentReceiver(this);

	Super::EndPlay(EndPlayReason);
}
```

关于GameStateBase的,引擎源码注释
``` txt
GameStateBase 是一个负责管理游戏全局状态的类，由 GameModeBase 生成。该类在客户端和服务器端均存在，并且实现了完全的复制功能。
```
## 3.创建ULyraLocalPlayer
很有意思这是个U类!
继承关系:
ULyraLocalPlayer->UCommonLocalPlayer->ULocalPlayer

ULocalPlayer源码注释:
``` txt
/**
* 在当前客户端/监听服务器上活跃的每个玩家都有一个“本地玩家”。
* 它会在不同地图之间保持活跃状态，在分屏/合作模式下可能会有多个此类玩家生成。
* 在专用服务器上不会生成任何此类玩家。*/
```
UCommonLocalPlayer:
``` cpp

UCLASS(MinimalAPI, config=Engine, transient)
class UCommonLocalPlayer : public ULocalPlayer
{
	GENERATED_BODY()

public:
	UE_API UCommonLocalPlayer();

	/** Called when the local player is assigned a player controller */
	/** 当本地玩家被分配到玩家控制器时会调用此函数 */
	DECLARE_MULTICAST_DELEGATE_TwoParams(FPlayerControllerSetDelegate, UCommonLocalPlayer* LocalPlayer, APlayerController* PlayerController);
	FPlayerControllerSetDelegate OnPlayerControllerSet;

	/** Called when the local player is assigned a player state */
	/** 当本地玩家被分配到特定玩家状态时会调用此函数 */
	DECLARE_MULTICAST_DELEGATE_TwoParams(FPlayerStateSetDelegate, UCommonLocalPlayer* LocalPlayer, APlayerState* PlayerState);
	FPlayerStateSetDelegate OnPlayerStateSet;

	/** Called when the local player is assigned a player pawn */
	/** 当本地玩家获得一个玩家棋子时会调用此函数 */
	DECLARE_MULTICAST_DELEGATE_TwoParams(FPlayerPawnSetDelegate, UCommonLocalPlayer* LocalPlayer, APawn* Pawn);
	FPlayerPawnSetDelegate OnPlayerPawnSet;

	UE_API FDelegateHandle CallAndRegister_OnPlayerControllerSet(FPlayerControllerSetDelegate::FDelegate Delegate);
	UE_API FDelegateHandle CallAndRegister_OnPlayerStateSet(FPlayerStateSetDelegate::FDelegate Delegate);
	UE_API FDelegateHandle CallAndRegister_OnPlayerPawnSet(FPlayerPawnSetDelegate::FDelegate Delegate);

public:
	/**
	* 用于获取投影所需的各种数据的辅助函数*
	* @参数	视口				视图客户端的视口
	* @参数	投影数据			用于填充投影数据的结构体
	* @参数	立体视图索引		使用立体显示时视图的索引
	* @返回 值为假的情况包括：没有视口，或者该 Actor 为 null 。
	* 
	*/
	UE_API virtual bool GetProjectionData(FViewport* Viewport, FSceneViewProjectionData& ProjectionData, int32 StereoViewIndex) const override;

	bool IsPlayerViewEnabled() const { return bIsPlayerViewEnabled; }
	void SetIsPlayerViewEnabled(bool bInIsPlayerViewEnabled) { bIsPlayerViewEnabled = bInIsPlayerViewEnabled; }

	//拿到主要用户界面布局.
	UE_API UPrimaryGameLayout* GetRootUILayout() const;

private:
	bool bIsPlayerViewEnabled = true;
};
```

ULyraLocalPlayer:
``` cpp
/**
 * ULyraLocalPlayer
 */
UCLASS(MinimalAPI)
class ULyraLocalPlayer : public UCommonLocalPlayer
{
	GENERATED_BODY()
	
public:

	UE_API ULyraLocalPlayer();
	
};
```
记得在DefaultEngine.ini中指定
``` ini
; 本地玩家所使用的类
LocalPlayerClassName=/Script/LyraGame.LyraLocalPlayer
```

## 4.创建LyraPlayerController
注意该类的继承关系
APlayerController->AModularPlayerController->ACommonPlayerController->

APlayerController源码注释:
``` txt
/**
玩家控制器由玩家使用，用于控制棋子。*
* 控制旋转（通过“获取控制旋转”方法访问），用于确定受控角色的瞄准方向。*
在联网游戏中，每个玩家操控的棋子都有对应的“玩家控制器”存在于服务器中，
并且也存在于控制该棋子的客户端设备上。但对于网络中其他地方由远程玩家操控的棋子，
其控制器则不会存在于客户端设备上。*
* @参考链接：https://docs.unrealengine.com/latest/INT/Gameplay/Framework/Controller/PlayerController/*/
```
AModularPlayerController源码注释:
在其源文件里面,注册为了接收者,注意发送的Ready事件的时机,并且也通知了对应的ControllerComponent
``` txt
/** 一个仅支持通过游戏功能插件进行扩展的最简类 */
```

ACommonPlayerController源码注释:
在用户控制室初始化的过程中触发UCommonLocalPlayer中的代理

ALyraPlayerController初始代码:
``` cpp
/**
 * ALyraPlayerController
 *
 *	The base player controller class used by this project.
 * 本项目所使用的基础玩家控制器类。
 */
UCLASS(MinimalAPI, Config = Game, Meta = (ShortTooltip = "The base player controller class used by this project."))
class ALyraPlayerController : public ACommonPlayerController
{
	GENERATED_BODY()
	
public:
	/*
	 * 构建函数
	 * 设置相机组件类
	 * 设置作弊组件类
	 */
	UE_API ALyraPlayerController(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	//获取玩家状态
	UFUNCTION(BlueprintCallable, Category = "Lyra|PlayerController")
	UE_API ALyraPlayerState* GetLyraPlayerState() const;

	//获取玩家HUD
	UFUNCTION(BlueprintCallable, Category = "Lyra|PlayerController")
	UE_API ALyraHUD* GetLyraHUD() const;
	
};


```


## 5.创建ALyraReplayPlayerController
该类继承自ALyraPlayerController

``` cpp

// A player controller used for replay capture and playback
// 用于录制和回放的玩家控制器
UCLASS()
class ALyraReplayPlayerController : public ALyraPlayerController
{
	GENERATED_BODY()

};



```


## 6.创建ALyraPlayerState
ALyraPlayerState->AModularPlayerState->APlayerState.
APlayerState:
``` txt
/**
* 在服务器（或独立游戏中）中，会为每个玩家创建一个“玩家状态”对象。
* 玩家状态会被复制到所有客户端，并包含与网络游戏相关的关于该玩家的信息，例如玩家姓名、得分等。
*/
```
AModularPlayerState:
这个类主要是做了转发,还有注册为接受者,发送Ready时间
``` cpp
/** Minimal class that supports extension by game feature plugins */
/** 一个仅支持通过游戏功能插件进行扩展的最简类 */
UCLASS(MinimalAPI, Blueprintable)
class AModularPlayerState : public APlayerState
{
	GENERATED_BODY()

public:
	//~ Begin AActor interface
	/** 在组件初始化之前被调用，仅在游戏过程中被调用 */
	UE_API virtual void PreInitializeComponents() override;
	
	/** 可重写的原生事件，用于表示此角色开始播放时的情况。*/
	UE_API virtual void BeginPlay() override;
	
	/** 可重载的函数，每当此角色从关卡中移除时会调用此函数 */
	UE_API virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;

	/** 将角色重置至初始状态 - 用于在不重新加载的情况下重新开始关卡。*/
	UE_API virtual void Reset() override;
	//~ End AActor interface

protected:
	//~ Begin APlayerState interface
	/** Copy properties which need to be saved in inactive PlayerState */
	/** 复制需要保存在非活动玩家状态中的属性 */
	UE_API virtual void CopyProperties(APlayerState* PlayerState);
	//~ End APlayerState interface
};

```

ALyraPlayerState:
``` cpp
/**
 * ALyraPlayerState
 *
 *	Base player state class used by this project.
 *	本项目所使用的基础玩家状态类。
 */
UCLASS(MinimalAPI, Config = Game)
class ALyraPlayerState : public AModularPlayerState
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

	//获取玩家控制器
	UFUNCTION(BlueprintCallable, Category = "Lyra|PlayerState")
	UE_API ALyraPlayerController* GetLyraPlayerController() const;

	
};
```

在Actor代码里面有一段关于初始化顺序的介绍 大概过一眼即可.
```
/**
* “Actor” 是一种基类，用于表示能够在关卡中放置或生成的对象。
* 一个“Actor”可以包含一系列“ActorComponent”，这些组件可用于控制对象的移动方式、渲染方式等。
* “Actor”的另一个主要功能是在游戏过程中通过网络复制其属性和函数调用。*
*
角色初始化包含多个步骤，以下是会调用的重要虚函数的执行顺序：
- UObject::PostLoad：对于静态放置在关卡中的角色，无论是在编辑器中还是在游戏过程中，都会调用普通的 UObject PostLoad 函数。
                      对于新生成的角色则不会调用此函数。
- UActorComponent:：OnComponentCreated：当在编辑器中或游戏过程中生成角色时，会为任何原生组件调用此函数。
                                        对于通过蓝图创建的组件，此函数会在该组件的构建过程中调用。
                                        对于从关卡加载的组件则不会调用此函数。
- AActor:：PreRegisterAllComponents：对于静态放置的角色和具有原生根组件的生成角色，现在会调用此函数。
                                     对于没有原生根组件的蓝图角色，这些注册函数会在构建过程中稍后调用。
- UActorComponent:：RegisterComponent：所有组件都会在编辑器和运行时进行注册，此函数会创建其物理/视觉表示。这些调用可能会分布在多个帧中，但总是在“预注册所有组件”之后进行。
这在“取消注册组件”操作将组件从游戏世界中移除之后的稍后阶段也可能被调用。
- AActor:：PostRegisterAllComponents：无论是在编辑器中还是在游戏过程中，都会为所有角色调用此函数，这是所有情况下都会被调用的最后一个函数。
- AActor:：PostActorCreated：当在编辑器中或游戏过程中创建角色时，此函数会在构造之前立即被调用。
  （此函数不会为从关卡加载的组件调用。）
- AActor:：UserConstructionScript：针对实现构建脚本的蓝图调用此函数。
- AActor:：OnConstruction：在执行构建操作的末尾调用此函数，它会调用蓝图构建脚本。
  （此函数在所有蓝图创建的组件完全创建并注册之后被调用。）
  （仅在游戏过程中会为生成的角色调用此函数，在编辑器中更改蓝图时可能会重新运行。）* - AActor::PreInitializeComponents：在对角色组件调用 InitializeComponent 之前被调用。
*                                    这种调用仅在游戏过程中以及某些编辑器预览窗口中进行。
* - UActorComponent::Activate：只有当组件的 bAutoActivate 属性设置为 true 时才会被调用。
*                              如果组件是手动激活的，也会稍后被调用。
* - UActorComponent:：InitializeComponent：只有当组件的 bWantsInitializeComponentSet 属性设置为 true 时才会被调用。
*                                         这种情况仅在游戏过程中出现一次。
* - AActor:：PostInitializeComponents：在角色组件初始化完成之后被调用，仅在游戏过程中以及某些编辑器预览窗口中进行。
* - AActor:：BeginPlay：当关卡开始计时时被调用，仅在实际游戏过程中进行。
*                      此操作通常在 PostInitializeComponents 之后立即进行，但在网络化或子角色中可能会延迟执行。*
* @参考链接：https://docs.unrealengine.com/Programming/UnrealArchitecture/Actors
* @参考链接：https://docs.unrealengine.com/Programming/UnrealArchitecture/Actors/ActorLifecycle
* @参考类：UActorComponent+
*/
```
## 7.创建LyraCharacter
ALyraCharacter->AModularCharacter->ACharacter
ACharacter:
``` txt
/**Lyra
* 这些角色被称为“兵卒”，它们具有网格结构、碰撞检测以及内置的移动逻辑。
* 它们负责玩家或人工智能与世界之间的所有物理交互，并且还实现了基本的网络通信和输入模型。
* 它们的设计适用于纵向呈现的玩家角色，能够通过“CharacterMovementComponent”在世界中行走、跳跃、飞行和游泳。*
* @参见 APawn、UCharacterMovementComponent
* @查看 https://docs.unrealengine.com/latest/INT/Gameplay/Framework/Pawn/Character/*/
```
AModularCharacter:
注册为接收者,并发送Read事件
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
ALyraCharacter:

``` cpp
/**
 * ALyraCharacter
 *
 *	The base character pawn class used by this project.
 *	Responsible for sending events to pawn components.
 *	New behavior should be added via pawn components when possible.
 *
 * 本项目所使用的基础角色兵种类。
 * 负责向兵种组件发送事件。
 * 在可能的情况下，应通过兵种组件添加新行为。
 *	
 */
UCLASS(MinimalAPI, Config = Game, Meta = (ShortTooltip = "The base character pawn class used by this project."))
class ALyraCharacter : public AModularCharacter
{
	GENERATED_BODY()

public:
	/*
	 * 构造函数
	 * 1.开启Tick
	 * 2.修改Mesh的Transform
	 * 3.修改移动组件参数
	 * 4.创建PawnExtComp管理组件拓展器
	 * 5.创建生命值组件
	 * 6.创建相机组件
	 *  
	 */
	UE_API ALyraCharacter(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	//获取玩家控制器
	UFUNCTION(BlueprintCallable, Category = "Lyra|Character")
	UE_API ALyraPlayerController* GetLyraPlayerController() const;

	//获取玩家状态类
	UFUNCTION(BlueprintCallable, Category = "Lyra|Character")
	UE_API ALyraPlayerState* GetLyraPlayerState() const;
	
};


```


ALyraCharacterWithAbilities
``` cpp
// ALyraCharacter typically gets the ability system component from the possessing player state
// This represents a character with a self-contained ability system component.
// ALyraCharacter通常会从拥有者的玩家状态中获取能力系统组件
// 这表示该角色拥有独立的能力系统组件。
UCLASS(MinimalAPI, Blueprintable)
class ALyraCharacterWithAbilities : public ALyraCharacter
{
	GENERATED_BODY()

public:
	/*
	 * 构造函数
	 * 1.创建GAS组件,设置网络同步模式
	 * 2.创建生命值组件
	 * 3.创建战斗组件
	 * 4.设置更新频率
	 */
	UE_API ALyraCharacterWithAbilities(const FObjectInitializer& ObjectInitializer);


};
```
## 8.创建LyraHUD
因为我们的UI是通过在HUD Layout的基础上进行推送的.所以这个HUD其实没啥用.
主要是为了添加拥有ASC的对象作为debug显示.
``` cpp

/**
 * ALyraHUD
 *
 *  Note that you typically do not need to extend or modify this class, instead you would
 *  use an "Add Widget" action in your experience to add a HUD layout and widgets to it
 *
 *  请注意，您通常无需对这个类进行扩展或修改，而是可以通过在您的体验中使用“添加控件”操作来添加 HUD 布局和控件到其中。
 *  
 *  This class exists primarily for debug rendering
 *  这个类的主要用途是用于调试渲染。
 */
UCLASS(Config = Game)
class ALyraHUD : public AHUD
{
	GENERATED_BODY()

public:
	//构造函数 关闭Tick
	ALyraHUD(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

protected:

	//~UObject interface
	virtual void PreInitializeComponents() override;
	//~End of UObject interface

	//~AActor interface
	virtual void BeginPlay() override;
	virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
	//~End of AActor interface

	//~AHUD interface
	/**
	 * 获取“显示调试信息”所涉及的目标列表
	 * 该列表是根据已启用的“显示调试信息”标志的上下文情况进行构建的。
	 * 
	 */
	virtual void GetDebugActorList(TArray<AActor*>& InOutList) override;
	//~End of AHUD interface
};
```
## 9.创建LyraGameSession
这个是会话的类.没有写更多的内容.
LyraGameSession->AGameSession

AGameSession:
``` txt
/**
它是一个针对特定游戏的接口封装器，位于会话接口的外部。当游戏代码需要与会话接口进行交互时，会调用此接口。
游戏会话仅存在于服务器端，用于运行在线游戏。
*/

```
LyraGameSession:
``` cpp

UCLASS(Config = Game)
class ALyraGameSession : public AGameSession
{
	GENERATED_BODY()

public:

	ALyraGameSession(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

protected:

/**
* 允许在线服务在命令行中通过 -auth_login 或 -auth_password 参数指定的情况下进行登录处理
* @返回值：如果登录正在进行，则返回 true，否则返回 false*/
	virtual bool ProcessAutoLogin() override;
	
	/** 当比赛开始时的处理操作 */
	virtual void HandleMatchHasStarted() override;

	/** 当匹配完成时的处理操作 */
	virtual void HandleMatchHasEnded() override;
};


```

## 10.关联到GameMode

在GameMode里面设置我们创建的自定义游戏基础类
``` cpp
ALyraGameMode::ALyraGameMode(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)

{	GameStateClass = ALyraGameState::StaticClass();
	GameSessionClass = ALyraGameSession::StaticClass();
	PlayerControllerClass = ALyraPlayerController::StaticClass();
	ReplaySpectatorPlayerControllerClass = ALyraReplayPlayerController::StaticClass();
	PlayerStateClass = ALyraPlayerState::StaticClass();
	DefaultPawnClass = ALyraCharacter::StaticClass();
	HUDClass = ALyraHUD::StaticClass();
}

```

然后在项目的Conent目录下创建BP_LyraGameMode.
在Project Settings_Map&Modes下指定我们的GamoMode即可.
其实这时候发现我们的B_LyraGameMode蓝图类复活了,是不是很神奇!


## 11.创建LyraGameViewportClient
``` txt
/**
 * UGameViewportClient
 * 游戏视口（FViewport）是一个针对特定平台的渲染、音频和输入子系统的高级抽象接口。
 * GameViewportClient 是引擎与游戏视口之间的接口。
 * 对于每个游戏实例，都会创建一个唯一的 GameViewportClient 实例。最终的译文：这个
 * 目前唯一一种可能的情况是，您可能会有一个引擎实例，但
 * 游戏的多个实例（因此会有多个 GameViewportClients）的情况是当
 * 您同时运行多个 PIE 窗口时才会出现。*
 * 职责：
 * 将输入事件传播至全局交互列表中*
 * @见 UGameViewportClient
 * 
 */
 ```

 ``` txt
// UCommonGameViewportClient: 常用用户界面视图首先会将输入重新导向用户界面。这是为了使常用用户界面能够进行输入的路由/处理。
```

``` cpp
// Lyra项目使用的视口类
UCLASS(BlueprintType)
class ULyraGameViewportClient : public UCommonGameViewportClient
{
	GENERATED_BODY()

public:
	ULyraGameViewportClient();

	virtual void Init(struct FWorldContext& WorldContext, UGameInstance* OwningGameInstance, bool bCreateNewAudioDevice = true) override;
};

```
记得在ini文件中指定:
``` ini

[/Script/Engine.Engine]
; 游戏视口所使用的类
GameViewportClientClassName=/Script/LyraGame.LyraGameViewportClient

```

## 12.创建LyraSettingsLocal
``` ini
/**
 * UGameUserSettings:
 * Stores user settings for a game (for example graphics and sound settings), with the ability to save and load to and from a file.
 *
 * 为游戏保存用户设置（例如图形和声音设置），并具备将设置保存至文件、从文件加载设置以及从文件中加载并应用设置的功能。
 * 
 */

```

``` cpp

UCLASS()
class ULyraSettingsLocal : public UGameUserSettings
{
	GENERATED_BODY()

public:

	ULyraSettingsLocal();

	static ULyraSettingsLocal* Get();

};

```
记得在ini文件中指定:
``` ini
; 用户游戏设置所使用的类
GameUserSettingsClassName=/Script/LyraGame.LyraSettingsLocal
```
## 总结
记得去编辑器下检查是否已完成更换.这个地方容易漏
本节的内容看似轻松,但是这一切都是建立在良好的UEC++基础上的.
我们只提及到了Modular层面.这一层面以前通常是我们项目的顶级父类层面.但是这个插件是基于多个项目抽取出来的一个层面.再往下才是我们的项目层面.
大家有时间的话应该多阅读引擎基类的代码,和使用流程.这样就可以避免很多bug的出现.
比如BeginPlay的蓝图和C++执行先后顺序?
玩家登录的初始化流程?
角色Possess执行的时机?
等等