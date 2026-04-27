# UE5_Lyra学习指南_018_玩家生成点
本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_018\_玩家生成点](#ue5_lyra学习指南_018_玩家生成点)
	- [概述](#概述)
	- [LyraPlayerStart](#lyraplayerstart)
		- [Tag容器](#tag容器)
		- [碰撞](#碰撞)
		- [宣称](#宣称)
	- [代码](#代码)
	- [总结](#总结)



## 概述
这是Lyra项目自定义的玩家出生点.在GameMode中选择玩家出生点时,必须使用该类!

## LyraPlayerStart
### Tag容器
它具有一个Tag的容器.它的父类也具有一个Tag的容器.  
这里不要搞混了
``` cpp
	/** Tags to identify this player start */
	/** 用于标识此玩家的标签开始 */
	UPROPERTY(EditAnywhere)
	FGameplayTagContainer StartPointTags;

```
```cpp

	/** Used when searching for which playerstart to use. */
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Object)
	FName PlayerStartTag;
```
### 碰撞
如果要使用该出生点来出生玩家,那么需要在使用前进行碰撞检定.  
简而言之.  
如果,完全不能生成,则返回Full
如果,直接可以生成,则返回Empty
如果,挪动一点位置,则返回Part
``` cpp
ELyraPlayerStartLocationOccupancy ALyraPlayerStart::GetLocationOccupancy(AController* const ControllerPawnToFit) const
{
	UWorld* const World = GetWorld();
	if (HasAuthority() && World)
	{
		if (AGameModeBase* AuthGameMode = World->GetAuthGameMode())
		{
			TSubclassOf<APawn> PawnClass = AuthGameMode->GetDefaultPawnClassForController(ControllerPawnToFit);
			const APawn* const PawnToFit = PawnClass ? GetDefault<APawn>(PawnClass) : nullptr;

			FVector ActorLocation = GetActorLocation();
			const FRotator ActorRotation = GetActorRotation();
			/** 如果演员在测试位置处会因受到阻碍物的阻挡而被迫移动，则返回 true 。 同时还会返回一个可能使测试位置不再受阻碍的调整建议值。*/
			if (!World->EncroachingBlockingGeometry(PawnToFit, ActorLocation, ActorRotation, nullptr))
			{
				// 可以完美的放进去
				return ELyraPlayerStartLocationOccupancy::Empty;
			}
			/**
			 * 尝试找到一个可接受且不会发生碰撞的位置来放置 TestActor，使其尽可能靠近 PlaceLocation。要求 PlaceLocation 必须是关卡内有效的位置。
			 * 如果找到了一个没有阻碍碰撞的合适位置，则返回 true，此时 PlaceLocation 会被更新为新的无碰撞位置。
			 * 如果未找到合适的位置，则返回 false，此时 PlaceLocation 不会改变。
			 * 
			 */
			else if (World->FindTeleportSpot(PawnToFit, ActorLocation, ActorRotation))
			{
				// 需要移动部分位置
				return ELyraPlayerStartLocationOccupancy::Partial;
			}
		}
	}
	// 被完全阻塞了
	return ELyraPlayerStartLocationOccupancy::Full;
}

```

### 宣称
当一个控制器选择该出生点生成玩家时,会将该出生点和控制器配对.  
同时设置一个定时器,当玩家出生后离开这里,将出生点重置!避免多个玩家选择同一出生点导致碰撞卡死!  
该定时器会Loop.只有当玩家彻底离开后,宣称才会重置.
``` cpp
bool ALyraPlayerStart::TryClaim(AController* OccupyingController)
{
	if (OccupyingController != nullptr && !IsClaimed())
	{
		ClaimingController = OccupyingController;
		if (UWorld* World = GetWorld())
		{
			// 这里需要有一个宣称判断
			World->GetTimerManager().SetTimer(ExpirationTimerHandle,
				FTimerDelegate::CreateUObject(this, &ALyraPlayerStart::CheckUnclaimed), ExpirationCheckInterval, true);
		}
		return true;
	}
	return false;
}
```

``` cpp
void ALyraPlayerStart::CheckUnclaimed()
{
	if (ClaimingController != nullptr && ClaimingController->GetPawn() != nullptr && GetLocationOccupancy(ClaimingController) == ELyraPlayerStartLocationOccupancy::Empty)
	{
		// 证明已经使用完毕啦!
		ClaimingController = nullptr;
		if (UWorld* World = GetWorld())
		{
			World->GetTimerManager().ClearTimer(ExpirationTimerHandle);
		}
	}
}


```



## 代码
LyraPlayerStart.h:
``` cpp
enum class ELyraPlayerStartLocationOccupancy
{
	// 空闲
	Empty,
	// 部分占用
	Partial,
	// 完全占用
	Full
};

/**
 * ALyraPlayerStart
 * 
 * Base player starts that can be used by a lot of modes.
 * 基础角色可被多种模式所使用。
 */
UCLASS(MinimalAPI, Config = Game)
class ALyraPlayerStart : public APlayerStart
{
	GENERATED_BODY()

public:
	UE_API ALyraPlayerStart(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	// 获取Tag
	const FGameplayTagContainer& GetGameplayTags() { return StartPointTags; }

	// 获取目标位置的是否可以放下Pawn的碰撞状态.
	UE_API ELyraPlayerStartLocationOccupancy GetLocationOccupancy(AController* const ControllerPawnToFit) const;

	/** Did this player start get claimed by a controller already? */
	// 是否被占用
	UE_API bool IsClaimed() const;

	/** If this PlayerStart was not claimed, claim it for ClaimingController */
	// 尝试占用
	UE_API bool TryClaim(AController* OccupyingController);

protected:
	/** Check if this PlayerStart is still claimed */
	/** 检查此玩家起点是否仍处于占用状态 */
	UE_API void CheckUnclaimed();

	/** The controller that claimed this PlayerStart */
	/** 承担此玩家起始点控制权的控制器 */
	UPROPERTY(Transient)
	TObjectPtr<AController> ClaimingController = nullptr;

	/** Interval in which we'll check if this player start is not colliding with anyone anymore */
	/** 我们将在此间隔时间内检查此玩家的移动是否不再与任何人发生碰撞 */
	UPROPERTY(EditDefaultsOnly, Category = "Player Start Claiming")
	float ExpirationCheckInterval = 1.f;

	/** Tags to identify this player start */
	/** 用于标识此玩家的标签开始 */
	UPROPERTY(EditAnywhere)
	FGameplayTagContainer StartPointTags;

	/** Handle to track expiration recurring timer */
	/** 用于跟踪定期过期定时器的标识符 */
	FTimerHandle ExpirationTimerHandle;
};

#undef UE_API


```

## 总结
这个自定义的玩家出生点主要用于实现不同队伍,不同出生点的类,在其他地方被引用使用.  
比如:UTDM_PlayerSpawningManagmentComponent,ULyraPlayerSpawningManagerComponent
同时要注意如何去项目中,灵活的检测这个出生点是不是用成引擎原生的了!
比如WorldSettings中去CheckError.通过Actor生成时判定.关卡流送进世界时判定等!
