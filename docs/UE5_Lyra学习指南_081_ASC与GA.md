# UE5_Lyra学习指南_081_ASC与GA

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_081\_ASC与GA](#ue5_lyra学习指南_081_asc与ga)
	- [概述](#概述)
	- [能力的来源](#能力的来源)
	- [按键激活流程](#按键激活流程)
		- [同步按键输入事件](#同步按键输入事件)
	- [全局ASC的注册与停用](#全局asc的注册与停用)
		- [停用](#停用)
		- [注册](#注册)
		- [注册的调用位置](#注册的调用位置)
			- [PlayerState](#playerstate)
			- [GameState](#gamestate)
			- [PawnExtension](#pawnextension)
			- [LyraCharacterWithAbilities](#lyracharacterwithabilities)
	- [尝试激活生成时能力](#尝试激活生成时能力)
	- [能力的激活](#能力的激活)
		- [激活流程](#激活流程)
		- [激活的方式](#激活的方式)
		- [可激活判断](#可激活判断)
		- [TagRelationShip的嵌入](#tagrelationship的嵌入)
		- [激活后的分组](#激活后的分组)
			- [介入时机](#介入时机)
		- [能力激活失败](#能力激活失败)
		- [能力组的切换](#能力组的切换)
		- [通过能力组互斥取消能力](#通过能力组互斥取消能力)
	- [能力的消耗](#能力的消耗)
	- [能力相机](#能力相机)
	- [用于作弊时施加动态标签](#用于作弊时施加动态标签)
	- [总结](#总结)



## 概述
本节主要拆解讲解Lyra中的ASC.
## 能力的来源

注意通过GameFeature或者武器组件同样能授予新的能力集.
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
	for (const ULyraAbilitySet* AbilitySet : PawnData->AbilitySets)
	{
		if (AbilitySet)
		{
			AbilitySet->GiveToAbilitySystem(AbilitySystemComponent, nullptr);
		}
	}
	// 发送框架事件 GAS注册完毕!
	UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(this, NAME_LyraAbilityReady);
	
	/** 强制将角色信息更新至客户端/演示网络驱动程序 */
	ForceNetUpdate();
}


```

``` cpp

void ULyraAbilitySet::GiveToAbilitySystem(ULyraAbilitySystemComponent* LyraASC, FLyraAbilitySet_GrantedHandles* OutGrantedHandles, UObject* SourceObject) const
{
	check(LyraASC);

	if (!LyraASC->IsOwnerActorAuthoritative())
	{
		// Must be authoritative to give or take ability sets.
		return;
	}
	
	// Grant the attribute sets.
	for (int32 SetIndex = 0; SetIndex < GrantedAttributes.Num(); ++SetIndex)
	{
		const FLyraAbilitySet_AttributeSet& SetToGrant = GrantedAttributes[SetIndex];

		if (!IsValid(SetToGrant.AttributeSet))
		{
			UE_LOG(LogLyraAbilitySystem, Error, TEXT("GrantedAttributes[%d] on ability set [%s] is not valid"), SetIndex, *GetNameSafe(this));
			continue;
		}

		UAttributeSet* NewSet = NewObject<UAttributeSet>(LyraASC->GetOwner(), SetToGrant.AttributeSet);
		LyraASC->AddAttributeSetSubobject(NewSet);

		if (OutGrantedHandles)
		{
			OutGrantedHandles->AddAttributeSet(NewSet);
		}
	}

	// Grant the gameplay abilities.
	for (int32 AbilityIndex = 0; AbilityIndex < GrantedGameplayAbilities.Num(); ++AbilityIndex)
	{
		const FLyraAbilitySet_GameplayAbility& AbilityToGrant = GrantedGameplayAbilities[AbilityIndex];

		if (!IsValid(AbilityToGrant.Ability))
		{
			UE_LOG(LogLyraAbilitySystem, Error, TEXT("GrantedGameplayAbilities[%d] on ability set [%s] is not valid."), AbilityIndex, *GetNameSafe(this));
			continue;
		}

		ULyraGameplayAbility* AbilityCDO = AbilityToGrant.Ability->GetDefaultObject<ULyraGameplayAbility>();

		FGameplayAbilitySpec AbilitySpec(AbilityCDO, AbilityToGrant.AbilityLevel);
		AbilitySpec.SourceObject = SourceObject;
		// 这个地方很重要!!!!
		AbilitySpec.GetDynamicSpecSourceTags().AddTag(AbilityToGrant.InputTag);

		const FGameplayAbilitySpecHandle AbilitySpecHandle = LyraASC->GiveAbility(AbilitySpec);

		if (OutGrantedHandles)
		{
			OutGrantedHandles->AddAbilitySpecHandle(AbilitySpecHandle);
		}
	}

	// Grant the gameplay effects.
	for (int32 EffectIndex = 0; EffectIndex < GrantedGameplayEffects.Num(); ++EffectIndex)
	{
		const FLyraAbilitySet_GameplayEffect& EffectToGrant = GrantedGameplayEffects[EffectIndex];

		if (!IsValid(EffectToGrant.GameplayEffect))
		{
			UE_LOG(LogLyraAbilitySystem, Error, TEXT("GrantedGameplayEffects[%d] on ability set [%s] is not valid"), EffectIndex, *GetNameSafe(this));
			continue;
		}

		const UGameplayEffect* GameplayEffect = EffectToGrant.GameplayEffect->GetDefaultObject<UGameplayEffect>();
		const FActiveGameplayEffectHandle GameplayEffectHandle = LyraASC->ApplyGameplayEffectToSelf(GameplayEffect, EffectToGrant.EffectLevel, LyraASC->MakeEffectContext());

		if (OutGrantedHandles)
		{
			OutGrantedHandles->AddGameplayEffectHandle(GameplayEffectHandle);
		}
	}
}

```

## 按键激活流程
由增强型输入系统传入输入句柄
``` cpp
void ULyraHeroComponent::Input_AbilityInputTagPressed(FGameplayTag InputTag)
{
	if (const APawn* Pawn = GetPawn<APawn>())
	{
		if (const ULyraPawnExtensionComponent* PawnExtComp = ULyraPawnExtensionComponent::FindPawnExtensionComponent(Pawn))
		{
			if (ULyraAbilitySystemComponent* LyraASC = PawnExtComp->GetLyraAbilitySystemComponent())
			{
				LyraASC->AbilityInputTagPressed(InputTag);
			}
		}	
	}
}
```


``` cpp
	// 根据增强型输入系统通过Hero组件传递过来的输入资产Tag来添加按下和释放的句柄
	UE_API void AbilityInputTagPressed(const FGameplayTag& InputTag);
	UE_API void AbilityInputTagReleased(const FGameplayTag& InputTag);


```
``` cpp
void ULyraAbilitySystemComponent::AbilityInputTagPressed(const FGameplayTag& InputTag)
{
	if (InputTag.IsValid())
	{
		for (const FGameplayAbilitySpec& AbilitySpec : ActivatableAbilities.Items)
		{
			// 注意这个GetDynamicSpecSourceTags,这个是在注册能力的时候需要传入的
			if (AbilitySpec.Ability && (AbilitySpec.GetDynamicSpecSourceTags().HasTagExact(InputTag)))
			{
				InputPressedSpecHandles.AddUnique(AbilitySpec.Handle);
				InputHeldSpecHandles.AddUnique(AbilitySpec.Handle);
			}
		}
	}
}

void ULyraAbilitySystemComponent::AbilityInputTagReleased(const FGameplayTag& InputTag)
{
	if (InputTag.IsValid())
	{
		for (const FGameplayAbilitySpec& AbilitySpec : ActivatableAbilities.Items)
		{
			// 这个Tag很重要GetDynamicSpecSourceTags.需要在能力注册时就给它注册进去
			if (AbilitySpec.Ability && (AbilitySpec.GetDynamicSpecSourceTags().HasTagExact(InputTag)))
			{
				InputReleasedSpecHandles.AddUnique(AbilitySpec.Handle);
				InputHeldSpecHandles.Remove(AbilitySpec.Handle);
			}
		}
	}
}
```

由控制器处理输入的句柄
``` cpp
void ALyraPlayerController::PostProcessInput(const float DeltaTime, const bool bGamePaused)
{
	if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
	{
		LyraASC->ProcessAbilityInput(DeltaTime, bGamePaused);
	}

	Super::PostProcessInput(DeltaTime, bGamePaused);
}

```

句柄的处理
``` cpp
void ULyraAbilitySystemComponent::ProcessAbilityInput(float DeltaTime, bool bGamePaused)
{
	// 是否有阻塞能力输入的tag
	if (HasMatchingGameplayTag(TAG_Gameplay_AbilityInputBlocked))
	{
		ClearAbilityInput();
		return;
	}

	static TArray<FGameplayAbilitySpecHandle> AbilitiesToActivate;
	AbilitiesToActivate.Reset();

	//@TODO: See if we can use FScopedServerAbilityRPCBatcher ScopedRPCBatcher in some of these loops
	// 待办事项：查看是否可以在这些循环中使用 FScopedServerAbilityRPCBatcher ScopedRPCBatcher

	//
	// Process all abilities that activate when the input is held.
	// 处理所有在输入保持按住状态时激活的能力。
	//
	for (const FGameplayAbilitySpecHandle& SpecHandle : InputHeldSpecHandles)
	{
		if (const FGameplayAbilitySpec* AbilitySpec = FindAbilitySpecFromHandle(SpecHandle))
		{
			if (AbilitySpec->Ability && !AbilitySpec->IsActive())
			{
				const ULyraGameplayAbility* LyraAbilityCDO = Cast<ULyraGameplayAbility>(AbilitySpec->Ability);
				if (LyraAbilityCDO && LyraAbilityCDO->GetActivationPolicy() == ELyraAbilityActivationPolicy::WhileInputActive)
				{
					AbilitiesToActivate.AddUnique(AbilitySpec->Handle);
				}
			}
		}
	}

	//
	// Process all abilities that had their input pressed this frame.
	// 处理本帧中输入被按下的所有能力。
	//
	for (const FGameplayAbilitySpecHandle& SpecHandle : InputPressedSpecHandles)
	{
		if (FGameplayAbilitySpec* AbilitySpec = FindAbilitySpecFromHandle(SpecHandle))
		{
			if (AbilitySpec->Ability)
			{
				AbilitySpec->InputPressed = true;

				if (AbilitySpec->IsActive())
				{
					// Ability is active so pass along the input event.
					// 能力已激活，因此将输入事件传递出去。
					// 这里很重要 如果没有正确执行的话 在GA_Hero_Jump里面无法正确的监听到Wait input press/Wait input Release这俩个节点
					AbilitySpecInputPressed(*AbilitySpec);
				}
				else
				{
					// 能力还没有激活,所以等下就去激活
					
					const ULyraGameplayAbility* LyraAbilityCDO = Cast<ULyraGameplayAbility>(AbilitySpec->Ability);

					if (LyraAbilityCDO && LyraAbilityCDO->GetActivationPolicy() == ELyraAbilityActivationPolicy::OnInputTriggered)
					{
						AbilitiesToActivate.AddUnique(AbilitySpec->Handle);
					}
				}
			}
		}
	}

	//
	// Try to activate all the abilities that are from presses and holds.
	// We do it all at once so that held inputs don't activate the ability
	// and then also send a input event to the ability because of the press.
	//

	//
	// 尝试激活所有通过按压和保持操作而获得的技能。
	// 我们一次性完成所有操作，这样保持状态下的输入就不会激活该技能，
	// 同时还会因为按压操作而向该技能发送输入事件。
	// //
	for (const FGameplayAbilitySpecHandle& AbilitySpecHandle : AbilitiesToActivate)
	{
		TryActivateAbility(AbilitySpecHandle);
	}

	//
	// Process all abilities that had their input released this frame.
	// 处理本帧中已释放输入的全部能力。
	//
	for (const FGameplayAbilitySpecHandle& SpecHandle : InputReleasedSpecHandles)
	{
		if (FGameplayAbilitySpec* AbilitySpec = FindAbilitySpecFromHandle(SpecHandle))
		{
			if (AbilitySpec->Ability)
			{
				AbilitySpec->InputPressed = false;

				if (AbilitySpec->IsActive())
				{
					// Ability is active so pass along the input event.
					// 能力已激活，因此将输入事件传递出去。
					// 这里很重要 如果没有正确执行的话 在GA_Hero_Jump里面无法正确的监听到Wait input press/Wait input Release这俩个节点
					AbilitySpecInputReleased(*AbilitySpec);
				}
			}
		}
	}

	//
	// Clear the cached ability handles.
	// 清除缓存中的能力句柄。
	//
	InputPressedSpecHandles.Reset();
	InputReleasedSpecHandles.Reset();

	// 注意这里不需要管InputHeldSpecHandles,它是由按键释放的时候去容器中进行移除.
	// 这三个容器名字很像不要搞混了!!!
}



```
``` cpp
void ULyraAbilitySystemComponent::ClearAbilityInput()
{
	InputPressedSpecHandles.Reset();
	InputReleasedSpecHandles.Reset();
	InputHeldSpecHandles.Reset();
}


```

``` cpp
	// Handles to abilities that had their input pressed this frame.
	// 本帧中已按下输入的技能句柄。
	TArray<FGameplayAbilitySpecHandle> InputPressedSpecHandles;

	// Handles to abilities that had their input released this frame.
	// 当前帧中已解除输入的技能的处理指针。
	TArray<FGameplayAbilitySpecHandle> InputReleasedSpecHandles;

	// Handles to abilities that have their input held.
	// 持有输入操作的技能的处理程序。
	TArray<FGameplayAbilitySpecHandle> InputHeldSpecHandles;


```
``` cpp
ULyraAbilitySystemComponent::ULyraAbilitySystemComponent(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
{
	InputPressedSpecHandles.Reset();
	InputReleasedSpecHandles.Reset();
	InputHeldSpecHandles.Reset();

	FMemory::Memset(ActivationGroupCounts, 0, sizeof(ActivationGroupCounts));
}
```
### 同步按键输入事件
``` cpp

void ULyraAbilitySystemComponent::AbilitySpecInputPressed(FGameplayAbilitySpec& Spec)
{
	Super::AbilitySpecInputPressed(Spec);

	// We don't support UGameplayAbility::bReplicateInputDirectly.
	// Use replicated events instead so that the WaitInputPress ability task works.
	// 我们不支持 UGameplayAbility::bReplicateInputDirectly 这个特性。
	// 请改用复制事件的方式，以便“等待输入按下”能力任务能够正常运行。
	if (Spec.IsActive())
	{
PRAGMA_DISABLE_DEPRECATION_WARNINGS
		const UGameplayAbility* Instance = Spec.GetPrimaryInstance();
		FPredictionKey OriginalPredictionKey = Instance ? Instance->GetCurrentActivationInfo().GetActivationPredictionKey() : Spec.ActivationInfo.GetActivationPredictionKey();
PRAGMA_ENABLE_DEPRECATION_WARNINGS

		// Invoke the InputPressed event. This is not replicated here. If someone is listening, they may replicate the InputPressed event to the server.
		// 调用“输入按下”事件。此操作在此处未作详细说明。若有人在监听，则可将“输入按下”事件发送至服务器。
		InvokeReplicatedEvent(EAbilityGenericReplicatedEvent::InputPressed, Spec.Handle, OriginalPredictionKey);
	}
}

void ULyraAbilitySystemComponent::AbilitySpecInputReleased(FGameplayAbilitySpec& Spec)
{
	Super::AbilitySpecInputReleased(Spec);

	// We don't support UGameplayAbility::bReplicateInputDirectly.
	// Use replicated events instead so that the WaitInputRelease ability task works.
	// 我们不支持 UGameplayAbility::bReplicateInputDirectly 这个特性。
	// 请改用复制事件的方式，以便“等待输入释放”能力任务能够正常运行。
	if (Spec.IsActive())
	{
PRAGMA_DISABLE_DEPRECATION_WARNINGS
		const UGameplayAbility* Instance = Spec.GetPrimaryInstance();
		FPredictionKey OriginalPredictionKey = Instance ? Instance->GetCurrentActivationInfo().GetActivationPredictionKey() : Spec.ActivationInfo.GetActivationPredictionKey();
PRAGMA_ENABLE_DEPRECATION_WARNINGS

		// Invoke the InputReleased event. This is not replicated here. If someone is listening, they may replicate the InputReleased event to the server.
		// 调用“输入释放”事件。此操作在此处未作详细说明。若有人在监听，则可将“输入释放”事件发送至服务器。
		InvokeReplicatedEvent(EAbilityGenericReplicatedEvent::InputReleased, Spec.Handle, OriginalPredictionKey);
	}
}
```
## 全局ASC的注册与停用
### 停用
``` cpp
void ULyraAbilitySystemComponent::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
	if (ULyraGlobalAbilitySystem* GlobalAbilitySystem = UWorld::GetSubsystem<ULyraGlobalAbilitySystem>(GetWorld()))
	{
		GlobalAbilitySystem->UnregisterASC(this);
	}

	Super::EndPlay(EndPlayReason);
}

```
### 注册


``` cpp

void ULyraAbilitySystemComponent::InitAbilityActorInfo(AActor* InOwnerActor, AActor* InAvatarActor)
{
	/**
	 *	FGameplayAbilityActorInfo
	 * 与使用某种能力的演员相关联的缓存数据。
	 *  - 在 InitFromActor 方法中从一个 AActor* 初始化而来。
	 *  - 能力利用此信息来确定要作用于的演员对象。例如，而不是与特定的演员类绑定。
	 *  - 这些通常以指针的形式传递，以支持多态性。
	 *  - 项目可以重写 UAbilitySystemGlobals:：AllocAbilityActorInfo 来覆盖创建的默认结构类型。*
	*/
	/**
	 *  AbilityActorInfo
	 *  存储有关拥有者角色的相关缓存数据，这些数据是能力类频繁需要访问的（如移动组件、网格组件、动画实例等）
	 *  
	 */
	FGameplayAbilityActorInfo* ActorInfo = AbilityActorInfo.Get();
	check(ActorInfo);
	check(InOwnerActor);

	// 是否是产生了替身的更替
	const bool bHasNewPawnAvatar = Cast<APawn>(InAvatarActor) && (InAvatarActor != ActorInfo->AvatarActor);

	// 先执行父类的
	Super::InitAbilityActorInfo(InOwnerActor, InAvatarActor);

	// 因为有替身的切换, 所以我们项目自定义的ASC流程需要重新初始化
	if (bHasNewPawnAvatar)
	{

		/**
		 * ActivatableAbilities
		 * 我们能够激活的能力。
		 * - 这将包括非实例化能力的 CDO 以及每次执行时的实例化能力。
		 * - 角色实例化能力将是实际的实例（而非 CDO）*
		 * 这个数组并非是系统正常运行所必需的。它只是一个便于“赋予角色能力”的辅助功能。但能力也可以独立于“能力系统组件”发挥作用。
		 * 例如，可以编写一种能力，使其能够作用于“静态网格角色”。只要该能力不需要实例化或其他任何能力系统组件所能提供的功能，那么它就无需该组件就能正常运行。
		 * 
		 */
		
		
		// Notify all abilities that a new pawn avatar has been set
		// 通知所有能力，新的兵种形象已设定完成
		for (const FGameplayAbilitySpec& AbilitySpec : ActivatableAbilities.Items)
		{
			// 禁用过期警告
PRAGMA_DISABLE_DEPRECATION_WARNINGS
			// 对于GA而言NonInstanced这种实例化类型已经过期了,请不要再使用
			ensureMsgf(AbilitySpec.Ability && AbilitySpec.Ability->GetInstancingPolicy() != EGameplayAbilityInstancingPolicy::NonInstanced,
				TEXT("InitAbilityActorInfo: All Abilities should be Instanced (NonInstanced is being deprecated due to usability issues)."));
PRAGMA_ENABLE_DEPRECATION_WARNINGS
	
			TArray<UGameplayAbility*> Instances = AbilitySpec.GetAbilityInstances();
			for (UGameplayAbility* AbilityInstance : Instances)
			{
				ULyraGameplayAbility* LyraAbilityInstance = Cast<ULyraGameplayAbility>(AbilityInstance);
				
				if (LyraAbilityInstance)
				{
					// Ability instances may be missing for replays
					// 回放中可能存在能力实例缺失的情况
					LyraAbilityInstance->OnPawnAvatarSet();
				}
			}
		}

		// Register with the global system once we actually have a pawn avatar. We wait until this time since some globally-applied effects may require an avatar.
		// 当我们真正拥有角色形象时，便向全局系统进行注册。我们在此时才进行注册，因为某些全局生效的效果可能需要角色形象的存在。
		if (ULyraGlobalAbilitySystem* GlobalAbilitySystem = UWorld::GetSubsystem<ULyraGlobalAbilitySystem>(GetWorld()))
		{
			GlobalAbilitySystem->RegisterASC(this);
		}

		// 重新初始化动画蓝图 因为我们绑定的ASC变了 所以绑定的Tag来源变了
		if (ULyraAnimInstance* LyraAnimInst = Cast<ULyraAnimInstance>(ActorInfo->GetAnimInstance()))
		{
			LyraAnimInst->InitializeWithAbilitySystem(this);
		}

		// 激活那些pawn生成是需要激活的能力
		TryActivateAbilitiesOnSpawn();
	}
}


```



### 注册的调用位置
#### PlayerState
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
#### GameState
``` cpp
void ALyraGameState::PostInitializeComponents()
{
	Super::PostInitializeComponents();

	check(AbilitySystemComponent);

	// 初始化ASC的组件信息
	AbilitySystemComponent->InitAbilityActorInfo(/*Owner=*/ this, /*Avatar=*/ this);
}


```

#### PawnExtension
``` cpp
void ULyraPawnExtensionComponent::InitializeAbilitySystem(ULyraAbilitySystemComponent* InASC, AActor* InOwnerActor)
{
	check(InASC);
	check(InOwnerActor);

	if (AbilitySystemComponent == InASC)
	{
		// The ability system component hasn't changed.
		// 能力系统组件并未发生任何变化。
		return;
	}

	if (AbilitySystemComponent)
	{
		// Clean up the old ability system component.
		// 清理旧的能力系统组件。
		UninitializeAbilitySystem();
	}

	// 我们当前使用的替身
	APawn* Pawn = GetPawnChecked<APawn>();
	// ASC指向的替身,正常情况应该是空值
	AActor* ExistingAvatar = InASC->GetAvatarActor();

	UE_LOG(LogLyra, Verbose, TEXT("Setting up ASC [%s] on pawn [%s] owner [%s], existing [%s] "), *GetNameSafe(InASC), *GetNameSafe(Pawn), *GetNameSafe(InOwnerActor), *GetNameSafe(ExistingAvatar));
	// 这里是为了处理ASC居然在初始化之前就有了替身
	if ((ExistingAvatar != nullptr) && (ExistingAvatar != Pawn))
	{
		UE_LOG(LogLyra, Log, TEXT("Existing avatar (authority=%d)"), ExistingAvatar->HasAuthority() ? 1 : 0);

		// There is already a pawn acting as the ASC's avatar, so we need to kick it out
		// This can happen on clients if they're lagged: their new pawn is spawned + possessed before the dead one is removed

		// 已经有一个角色充当着 ASC 的化身，所以我们需要将其移除
		// 如果客户端出现延迟，这种情况就可能发生：新的角色会先生成并被附身，然后才移除掉死亡的角色
		ensure(!ExistingAvatar->HasAuthority());
		// 拿到之前的替身,将其正常释放掉,保证我们当前的ASC是干净的.
		if (ULyraPawnExtensionComponent* OtherExtensionComponent = FindPawnExtensionComponent(ExistingAvatar))
		{
			// 这里是另外一个角色拓展组件,但是它使用的ASC是我们即将使用的,所以需要由上一个角色拓展组件将其释放掉
			OtherExtensionComponent->UninitializeAbilitySystem();
		}
	}
	// 把当前使用的ASC缓存起来,这样我们就不需要到处去找了
	AbilitySystemComponent = InASC;
	// 初始化信息
	AbilitySystemComponent->InitAbilityActorInfo(InOwnerActor, Pawn);

	// 将我们的能力映射表设置上去
	if (ensure(PawnData))
	{
		InASC->SetTagRelationshipMapping(PawnData->TagRelationshipMapping);
	}
	// 我们的ASC 初始化好了 让生命值组件去完成属性绑定
	OnAbilitySystemInitialized.Broadcast();
}

```
#### LyraCharacterWithAbilities
``` cpp
void ALyraCharacterWithAbilities::PostInitializeComponents()
{
	Super::PostInitializeComponents();

	check(AbilitySystemComponent);
	// 初始化能力演员信息
	AbilitySystemComponent->InitAbilityActorInfo(this, this);
}

```
## 尝试激活生成时能力
``` cpp
void ULyraAbilitySystemComponent::TryActivateAbilitiesOnSpawn()
{
	/** 用于防止我们在遍历能力组件中的能力时，意外删除该组件中的能力 */
	ABILITYLIST_SCOPE_LOCK();

	for (const FGameplayAbilitySpec& AbilitySpec : ActivatableAbilities.Items)
	{
		if (const ULyraGameplayAbility* LyraAbilityCDO = Cast<ULyraGameplayAbility>(AbilitySpec.Ability))
		{
			LyraAbilityCDO->TryActivateAbilityOnSpawn(AbilityActorInfo.Get(), AbilitySpec);
		}
	}
}

```

``` cpp
void ULyraGameplayAbility::OnGiveAbility(const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilitySpec& Spec)
{
	Super::OnGiveAbility(ActorInfo, Spec);

	K2_OnAbilityAdded();

	TryActivateAbilityOnSpawn(ActorInfo, Spec);
}
```

``` cpp
void ULyraGameplayAbility::TryActivateAbilityOnSpawn(const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilitySpec& Spec) const
{
	// Try to activate if activation policy is on spawn.
	// 若激活策略设定为“按生成时自动激活”，则尝试进行激活操作。
	if (ActorInfo && !Spec.IsActive() && (ActivationPolicy == ELyraAbilityActivationPolicy::OnSpawn))
	{
		UAbilitySystemComponent* ASC = ActorInfo->AbilitySystemComponent.Get();
		const AActor* AvatarActor = ActorInfo->AvatarActor.Get();

		// If avatar actor is torn off or about to die, don't try to activate until we get the new one.
		// 如果角色头像被撕掉或者即将死亡，那么在获取新的头像之前不要尝试激活该角色。
		if (ASC && AvatarActor && !AvatarActor->GetTearOff() && (AvatarActor->GetLifeSpan() <= 0.0f))
		{
			const bool bIsLocalExecution = (NetExecutionPolicy == EGameplayAbilityNetExecutionPolicy::LocalPredicted) || (NetExecutionPolicy == EGameplayAbilityNetExecutionPolicy::LocalOnly);
			const bool bIsServerExecution = (NetExecutionPolicy == EGameplayAbilityNetExecutionPolicy::ServerOnly) || (NetExecutionPolicy == EGameplayAbilityNetExecutionPolicy::ServerInitiated);

			const bool bClientShouldActivate = ActorInfo->IsLocallyControlled() && bIsLocalExecution;
			const bool bServerShouldActivate = ActorInfo->IsNetAuthority() && bIsServerExecution;

			if (bClientShouldActivate || bServerShouldActivate)
			{
				ASC->TryActivateAbility(Spec.Handle);
			}
		}
	}
}


```

## 能力的激活
### 激活流程

``` cpp
/** 
 *	UAbilitySystemComponent	
 *
 *	A component to easily interface with the 3 aspects of the AbilitySystem:
 *	
 *	GameplayAbilities:
 *		-Provides a way to give/assign abilities that can be used (by a player or AI for example)
 *		-Provides management of instanced abilities (something must hold onto them)
 *		-Provides replication functionality
 *			-Ability state must always be replicated on the UGameplayAbility itself, but UAbilitySystemComponent provides RPC replication
 *			for the actual activation of abilities
 *			
 *	GameplayEffects:
 *		-Provides an FActiveGameplayEffectsContainer for holding active GameplayEffects
 *		-Provides methods for applying GameplayEffects to a target or to self
 *		-Provides wrappers for querying information in FActiveGameplayEffectsContainers (duration, magnitude, etc)
 *		-Provides methods for clearing/remove GameplayEffects
 *		
 *	GameplayAttributes
 *		-Provides methods for allocating and initializing attribute sets
 *		-Provides methods for getting AttributeSets
 *  
 */

``` 

``` cpp
/**
*	UAbilitySystemComponent*
* 一个能够方便地与能力系统的三个方面进行连接的组件：
* 游戏玩法能力：
* - 提供了一种方式来赋予/分配可使用的能力（例如，可以由玩家或人工智能使用）
* - 提供了对实例化能力的管理（必须有某种机制来保存这些能力）
* - 提供了复制功能
* - 能力状态必须始终在 UGameplayAbility 本身中进行复制，但 UAbilitySystemComponent 提供了 RPC 复制功能
* - 用于实际激活能力的激活功能由 UAbilitySystemComponent 提供
* 游戏玩法效果：
* - 提供一个 FActiveGameplayEffectsContainer 用于存放活跃的游戏玩法效果
* - 提供将游戏玩法效果应用到目标对象或自身的方法
* - 提供用于查询 FActiveGameplayEffectsContainer 中信息（持续时间、强度等）的包装器
* - 提供清除/移除游戏玩法效果的方法
* 游戏玩法属性
* - 提供了用于分配和初始化属性集的方法
* - 提供了获取属性集的方法
*/
```

``` cpp
/**
 * UGameplayAbility
 *	
 *	Abilities define custom gameplay logic that can be activated or triggered.
 *	
 *	The main features provided by the AbilitySystem for GameplayAbilities are: 
 *		-CanUse functionality:
 *			-Cooldowns
 *			-Costs (mana, stamina, etc)
 *			-etc
 *			
 *		-Replication support
 *			-Client/Server communication for ability activation
 *			-Client prediction for ability activation
 *			
 *		-Instancing support
 *			-Abilities can be non-instanced (native only)
 *			-Instanced per owner
 *			-Instanced per execution (default)
 *			
 *		-Basic, extendable support for:
 *			-Input binding
 *			-'Giving' abilities (that can be used) to actors
 *	
 *
 *	See GameplayAbility_Montage for an example of a non-instanced ability
 *		-Plays a montage and applies a GameplayEffect to its target while the montage is playing.
 *		-When finished, removes GameplayEffect.
 *	
 *	Note on replication support:
 *		-Non instanced abilities have limited replication support. 
 *			-Cannot have state (obviously) so no replicated properties
 *			-RPCs on the ability class are not possible either.
 *			
 *	To support state or event replication, an ability must be instanced. This can be done with the InstancingPolicy property.
 */

```

``` cpp
* UGameplayAbility*
* 能力定义了可激活或触发的自定义游戏逻辑。*
游戏能力系统所提供的“能力系统”主要功能包括：
- 可使用功能：
  - 冷却时间
  - 成本（法力值、体力值等）
  - 其他等等*
*		- 重复性支持
*			- 用于能力激活的客户端/服务器通信
*			- 客户端能力预测*
*		- 实例化支持
*			- 能力可以是非实例化的（仅限原生形式）
*			- 按所有者进行实例化
*			- 按执行次序进行实例化（默认设置）*
- 基本且可扩展的支持包括：
- 输入绑定
- 向角色“赋予”可用的能力*
*
* 参见 GameplayAbility_Montage 中的示例，了解非实例化能力的使用方法
* - 播放一段蒙太奇片段，并在片段播放期间对其目标应用一个游戏效果。
* - 播放结束后，移除游戏效果。*
* 关于复制支持的说明：
* - 非实例化能力的复制支持有限。
* - 由于无法拥有状态（这一点很明显），所以无法进行复制操作，也就没有可复制的属性。
* - 该能力类上的 RPC（远程过程调用）也不可行。*
* 为了实现状态或事件的复制，必须实例化一个“实例化策略”属性。
```
``` cpp
	// ----------------------------------------------------------------------------------------------------------------
	//
	//	The important functions:
	//	
	//		CanActivateAbility()	- const function to see if ability is activatable. Callable by UI etc
	//
	//		TryActivateAbility()	- Attempts to activate the ability. Calls CanActivateAbility(). Input events can call this directly.
	//								- Also handles instancing-per-execution logic and replication/prediction calls.
	//		
	//		CallActivateAbility()	- Protected, non virtual function. Does some boilerplate 'pre activate' stuff, then calls ActivateAbility()
	//
	//		ActivateAbility()		- What the abilities *does*. This is what child classes want to override.
	//	
	//		CommitAbility()			- Commits reources/cooldowns etc. ActivateAbility() must call this!
	//		
	//		CancelAbility()			- Interrupts the ability (from an outside source).
	//
	//		EndAbility()			- The ability has ended. This is intended to be called by the ability to end itself.
	//	
	// ----------------------------------------------------------------------------------------------------------------


```
###  激活的方式
``` cpp
/**
 * ELyraAbilityActivationPolicy
 *
 *	Defines how an ability is meant to activate.
 *	定义了该能力应如何被激活的方式。
 */
UENUM(BlueprintType)
enum class ELyraAbilityActivationPolicy : uint8
{
	// Try to activate the ability when the input is triggered.
	// 在输入被触发时尝试激活该能力。
	OnInputTriggered,

	// Continually try to activate the ability while the input is active.
	// 在输入处于激活状态时，持续尝试激活该能力。
	WhileInputActive,

	// Try to activate the ability when an avatar is assigned.
	// 当为角色分配了形象时，尝试激活该能力。
	OnSpawn
};
```

### 可激活判断
``` cpp
bool ULyraGameplayAbility::CanActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo,
	const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, FGameplayTagContainer* OptionalRelevantTags) const
{
	// 合法性判定
	if (!ActorInfo || !ActorInfo->AbilitySystemComponent.IsValid())
	{
		return false;
	}

	// 父类判定
	if (!Super::CanActivateAbility(Handle, ActorInfo, SourceTags, TargetTags, OptionalRelevantTags))
	{
		return false;
	}

	//@TODO Possibly remove after setting up tag relationships
	//@待办事项：在建立标签关系后可能需要删除此内容
	ULyraAbilitySystemComponent* LyraASC = CastChecked<ULyraAbilitySystemComponent>(ActorInfo->AbilitySystemComponent.Get());
	if (LyraASC->IsActivationGroupBlocked(ActivationGroup))
	{
		if (OptionalRelevantTags)
		{
			OptionalRelevantTags->AddTag(LyraGameplayTags::Ability_ActivateFail_ActivationGroup);
		}
		return false;
	}

	return true;
}


```

``` cpp
bool ULyraAbilitySystemComponent::IsActivationGroupBlocked(ELyraAbilityActivationGroup Group) const
{
	bool bBlocked = false;

	switch (Group)
	{
	case ELyraAbilityActivationGroup::Independent:
		// Independent abilities are never blocked.
		// 独立的能力永远不会受到阻碍。
		bBlocked = false;
		break;

	case ELyraAbilityActivationGroup::Exclusive_Replaceable:
	case ELyraAbilityActivationGroup::Exclusive_Blocking:
		// Exclusive abilities can activate if nothing is blocking.
		// 若无任何阻碍物阻挡，则专属技能即可激活。
		// 意思是当前只要没有阻塞的能力在生效就可以释放该能力.
		bBlocked = (ActivationGroupCounts[(uint8)ELyraAbilityActivationGroup::Exclusive_Blocking] > 0);
		break;

	default:
		checkf(false, TEXT("IsActivationGroupBlocked: Invalid ActivationGroup [%d]\n"), (uint8)Group);
		break;
	}

	return bBlocked;
}


```

``` cpp


bool ULyraGameplayAbility::DoesAbilitySatisfyTagRequirements(const UAbilitySystemComponent& AbilitySystemComponent,
	const FGameplayTagContainer* SourceTags, const FGameplayTagContainer* TargetTags, OUT FGameplayTagContainer* OptionalRelevantTags) const
{
	// Specialized version to handle death exclusion and AbilityTags expansion via ASC
	// 专门版本，用于通过 ASC 处理死亡排除和能力标签的扩展操作

	bool bBlocked = false;
	bool bMissing = false;

	// 获取到全局中 关于阻塞或者丢失使用的Tag
	UAbilitySystemGlobals& AbilitySystemGlobals = UAbilitySystemGlobals::Get();
	const FGameplayTag& BlockedTag = AbilitySystemGlobals.ActivateFailTagsBlockedTag;
	const FGameplayTag& MissingTag = AbilitySystemGlobals.ActivateFailTagsMissingTag;

	// Check if any of this ability's tags are currently blocked
	// 检查该能力的任何标签当前是否已被禁用/封锁
	/** 如果传入的标签中有任何被禁止的标签，则返回 true */
	if (AbilitySystemComponent.AreAbilityTagsBlocked(GetAssetTags()))
	{
		bBlocked = true;
	}

	const ULyraAbilitySystemComponent* LyraASC = Cast<ULyraAbilitySystemComponent>(&AbilitySystemComponent);
	static FGameplayTagContainer AllRequiredTags;
	static FGameplayTagContainer AllBlockedTags;
	
	/** 该能力仅在激活角色/组件同时具备以下所有标签的情况下方可激活 */
	AllRequiredTags = ActivationRequiredTags;
	/** 若激活该能力的行动者/组件带有以下任何标签，则该能力将被阻止生效 */
	AllBlockedTags = ActivationBlockedTags;

	// Expand our ability tags to add additional required/blocked tags
	// 扩展我们的能力标签，以添加更多必需/禁止的标签
	if (LyraASC)
	{
		// 调用ASC的TagRelationshipMapping
		LyraASC->GetAdditionalActivationTagRequirements(GetAssetTags(), AllRequiredTags, AllBlockedTags);
	}

	// Check to see the required/blocked tags for this ability
	// 检查查看此能力所需的/被禁止使用的标签
	if (AllBlockedTags.Num() || AllRequiredTags.Num())
	{
		static FGameplayTagContainer AbilitySystemComponentTags;
		
		AbilitySystemComponentTags.Reset();
		AbilitySystemComponent.GetOwnedGameplayTags(AbilitySystemComponentTags);

		if (AbilitySystemComponentTags.HasAny(AllBlockedTags))
		{
			if (OptionalRelevantTags && AbilitySystemComponentTags.HasTag(LyraGameplayTags::Status_Death))
			{
				// If player is dead and was rejected due to blocking tags, give that feedback
				// 如果玩家死亡且因阻挡标签而被拒绝，则给出相应的反馈信息
				OptionalRelevantTags->AddTag(LyraGameplayTags::Ability_ActivateFail_IsDead);
			}

			bBlocked = true;
		}
		// 如果没有满足激活要求 则miss
		if (!AbilitySystemComponentTags.HasAll(AllRequiredTags))
		{
			bMissing = true;
		}
	}
	// 自己的Tag要求
	if (SourceTags != nullptr)
	{
		if (SourceBlockedTags.Num() || SourceRequiredTags.Num())
		{
			if (SourceTags->HasAny(SourceBlockedTags))
			{
				bBlocked = true;
			}

			if (!SourceTags->HasAll(SourceRequiredTags))
			{
				bMissing = true;
			}
		}
	}
	//  目标的Tag要求
	if (TargetTags != nullptr)
	{
		if (TargetBlockedTags.Num() || TargetRequiredTags.Num())
		{
			if (TargetTags->HasAny(TargetBlockedTags))
			{
				bBlocked = true;
			}

			if (!TargetTags->HasAll(TargetRequiredTags))
			{
				bMissing = true;
			}
		}
	}
	//  如果被阻塞了 添加阻塞的失败标签
	if (bBlocked)
	{
		if (OptionalRelevantTags && BlockedTag.IsValid())
		{
			OptionalRelevantTags->AddTag(BlockedTag);
		}
		return false;
	}
	// 如果缺失了 添加缺失的失败标签
	if (bMissing)
	{
		if (OptionalRelevantTags && MissingTag.IsValid())
		{
			OptionalRelevantTags->AddTag(MissingTag);
		}
		return false;
	}
	// 如果都没有 则判定通过
	return true;
}
```

### TagRelationShip的嵌入
``` cpp
	/** Sets the current tag relationship mapping, if null it will clear it out */
	/** 设置当前的标签关系映射，如果为 null，则会将其清除 */
	UE_API void SetTagRelationshipMapping(ULyraAbilityTagRelationshipMapping* NewMapping);
	
	/** Looks at ability tags and gathers additional required and blocking tags */\
	/** 查看能力标签，并收集所需的附加标签和阻碍性标签 */
	// 通过TagRelationship来收集 来确认当前能力是否可以释放!
	UE_API void GetAdditionalActivationTagRequirements(const FGameplayTagContainer& AbilityTags,
		FGameplayTagContainer& OutActivationRequired,
		FGameplayTagContainer& OutActivationBlocked) const;


```


### 激活后的分组
``` cpp
	// Number of abilities running in each activation group.
	// 每个激活组中运行的能力数量。
	int32 ActivationGroupCounts[(uint8)ELyraAbilityActivationGroup::MAX];

```
``` cpp
/**
 * ELyraAbilityActivationGroup
 *
 *	Defines how an ability activates in relation to other abilities.
 * 定义了某一能力与其他能力激活方式之间的关系。
 *	
 */
UENUM(BlueprintType)
enum class ELyraAbilityActivationGroup : uint8
{
	// Ability runs independently of all other abilities.
	// 能力是独立于其他所有能力存在的。
	Independent,

	// Ability is canceled and replaced by other exclusive abilities.
	// 能力被取消，并被其他独特的能力所取代。
	Exclusive_Replaceable,

	// Ability blocks all other exclusive abilities from activating.
	// 该技能会阻止其他所有特殊能力的激活。
	Exclusive_Blocking,

	MAX	UMETA(Hidden)
};
```

#### 介入时机
``` cpp
void ULyraAbilitySystemComponent::NotifyAbilityActivated(const FGameplayAbilitySpecHandle Handle, UGameplayAbility* Ability)
{
	Super::NotifyAbilityActivated(Handle, Ability);

	if (ULyraGameplayAbility* LyraAbility = Cast<ULyraGameplayAbility>(Ability))
	{
		AddAbilityToActivationGroup(LyraAbility->GetActivationGroup(), LyraAbility);
	}
}
void ULyraAbilitySystemComponent::NotifyAbilityEnded(FGameplayAbilitySpecHandle Handle, UGameplayAbility* Ability, bool bWasCancelled)
{
	Super::NotifyAbilityEnded(Handle, Ability, bWasCancelled);

	if (ULyraGameplayAbility* LyraAbility = Cast<ULyraGameplayAbility>(Ability))
	{
		RemoveAbilityFromActivationGroup(LyraAbility->GetActivationGroup(), LyraAbility);
	}
}

```
``` cpp
void ULyraAbilitySystemComponent::AddAbilityToActivationGroup(ELyraAbilityActivationGroup Group, ULyraGameplayAbility* LyraAbility)
{
	// 合法性判定
	check(LyraAbility);
	check(ActivationGroupCounts[(uint8)Group] < INT32_MAX);

	// 该能力类别的计数增加
	ActivationGroupCounts[(uint8)Group]++;

	const bool bReplicateCancelAbility = false;

	switch (Group)
	{
	case ELyraAbilityActivationGroup::Independent:
		// Independent abilities do not cancel any other abilities.
		// 独立能力不会抵消任何其他能力。
		break;

	case ELyraAbilityActivationGroup::Exclusive_Replaceable:
	case ELyraAbilityActivationGroup::Exclusive_Blocking:
		// 如果当前的能力具有类别的替换或者阻塞 则需要去关闭之前处于对应类别的能力
		// 因为不可能是阻塞,因为是阻塞根本就没办法激活,所以只可能是取消类别,把之前的这个可替换的能力取消即可
		CancelActivationGroupAbilities(ELyraAbilityActivationGroup::Exclusive_Replaceable, LyraAbility, bReplicateCancelAbility);
		break;

	default:
		checkf(false, TEXT("AddAbilityToActivationGroup: Invalid ActivationGroup [%d]\n"), (uint8)Group);
		break;
	}

	// 阻塞和可替换 他们合计不能超过1!!!!!
	const int32 ExclusiveCount = ActivationGroupCounts[(uint8)ELyraAbilityActivationGroup::Exclusive_Replaceable] + ActivationGroupCounts[(uint8)ELyraAbilityActivationGroup::Exclusive_Blocking];
	if (!ensure(ExclusiveCount <= 1))
	{
		UE_LOG(LogLyraAbilitySystem, Error, TEXT("AddAbilityToActivationGroup: Multiple exclusive abilities are running."));
	}
}

void ULyraAbilitySystemComponent::RemoveAbilityFromActivationGroup(ELyraAbilityActivationGroup Group, ULyraGameplayAbility* LyraAbility)
{
	check(LyraAbility);
	check(ActivationGroupCounts[(uint8)Group] > 0);
	// 计数减少即可
	ActivationGroupCounts[(uint8)Group]--;
}
```

### 能力激活失败
``` cpp

void ULyraAbilitySystemComponent::NotifyAbilityFailed(const FGameplayAbilitySpecHandle Handle,
	UGameplayAbility* Ability, const FGameplayTagContainer& FailureReason)
{
	Super::NotifyAbilityFailed(Handle, Ability, FailureReason);

	if (APawn* Avatar = Cast<APawn>(GetAvatarActor()))
	{
		/**
		 *IsSupportedForNetworking
		 *	我们只能复制以下内容的引用：
		 *	- CDO 和数据资产（例如，静态、非实例化的游戏能力）
		 *	- 正在实例化的且将被在客户端创建的那些能力。*
	 	 *	其他情况下则不予以支持，而是会在客户端重新生成。
		 *	
		 */
		// 如果这个Pawn不是本地控制的,但是这个能力又是需要网络的.那么我们就手动一个RPC通知出去
		if (!Avatar->IsLocallyControlled() && Ability->IsSupportedForNetworking())
		{
			ClientNotifyAbilityFailed(Ability, FailureReason);
			return;
		}
	}

	HandleAbilityFailed(Ability, FailureReason);
}

```
``` cpp

void ULyraAbilitySystemComponent::HandleAbilityFailed(const UGameplayAbility* Ability, const FGameplayTagContainer& FailureReason)
{
	//UE_LOG(LogLyraAbilitySystem, Warning, TEXT("Ability %s failed to activate (tags: %s)"), *GetPathNameSafe(Ability), *FailureReason.ToString());

	// 转发给GA处理
	if (const ULyraGameplayAbility* LyraAbility = Cast<const ULyraGameplayAbility>(Ability))
	{
		LyraAbility->OnAbilityFailedToActivate(FailureReason);
	}	
}
```
``` cpp
	// 接收ASC的转发 能力激活失败了
	void OnAbilityFailedToActivate(const FGameplayTagContainer& FailedReason) const
	{
		NativeOnAbilityFailedToActivate(FailedReason);
		ScriptOnAbilityFailedToActivate(FailedReason);
	}

```
``` cpp
void ULyraGameplayAbility::NativeOnAbilityFailedToActivate(const FGameplayTagContainer& FailedReason) const
{
	bool bSimpleFailureFound = false;
	for (FGameplayTag Reason : FailedReason)
	{
		if (!bSimpleFailureFound)
		{
			if (const FText* pUserFacingMessage = FailureTagToUserFacingMessages.Find(Reason))
			{
				FLyraAbilitySimpleFailureMessage Message;
				Message.PlayerController = GetActorInfo().PlayerController.Get();
				Message.FailureTags = FailedReason;
				Message.UserFacingReason = *pUserFacingMessage;

				UGameplayMessageSubsystem& MessageSystem = UGameplayMessageSubsystem::Get(GetWorld());
				MessageSystem.BroadcastMessage(TAG_ABILITY_SIMPLE_FAILURE_MESSAGE, Message);
				bSimpleFailureFound = true;
			}
		}
		
		if (UAnimMontage* pMontage = FailureTagToAnimMontage.FindRef(Reason))
		{
			FLyraAbilityMontageFailureMessage Message;
			Message.PlayerController = GetActorInfo().PlayerController.Get();
			Message.AvatarActor = GetActorInfo().AvatarActor.Get();
			Message.FailureTags = FailedReason;
			Message.FailureMontage = pMontage;

			UGameplayMessageSubsystem& MessageSystem = UGameplayMessageSubsystem::Get(GetWorld());
			MessageSystem.BroadcastMessage(TAG_ABILITY_PLAY_MONTAGE_FAILURE_MESSAGE, Message);
		}
	}
}

```
### 能力组的切换
``` cpp
	// Returns true if the requested activation group is a valid transition.
	// 若所请求的激活组为有效的转换，则返回 true 。
	// 仅在死亡能力激活时中调用
	UFUNCTION(BlueprintCallable, BlueprintPure = false, Category = "Lyra|Ability", Meta = (ExpandBoolAsExecs = "ReturnValue"))
	UE_API bool CanChangeActivationGroup(ELyraAbilityActivationGroup NewGroup) const;

	// Tries to change the activation group.  Returns true if it successfully changed.
	// 尝试更改激活组。若成功更改则返回 true 。
	// 仅在死亡能力激活中调用 主要时为了阻塞其他的可替换和阻塞能力
	UFUNCTION(BlueprintCallable, BlueprintPure = false, Category = "Lyra|Ability", Meta = (ExpandBoolAsExecs = "ReturnValue"))
	UE_API bool ChangeActivationGroup(ELyraAbilityActivationGroup NewGroup);

```

``` cpp
bool ULyraGameplayAbility::CanChangeActivationGroup(ELyraAbilityActivationGroup NewGroup) const
{
	// 实例化策略判定
	if (!IsInstantiated() || !IsActive())
	{
		return false;
	}

	// 如果新旧策略组都一致 则可以过渡
	if (ActivationGroup == NewGroup)
	{
		return true;
	}

	ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponentFromActorInfo();
	check(LyraASC);
	// 如果GA是非阻塞的,但是ASC当前的阻塞通道里面有阻塞技能,所以我们不能过渡
	if ((ActivationGroup != ELyraAbilityActivationGroup::Exclusive_Blocking) && LyraASC->IsActivationGroupBlocked(NewGroup))
	{
		// This ability can't change groups if it's blocked (unless it is the one doing the blocking).
		// 若该能力被阻挡，则无法改变组别（除非是进行阻挡操作的一方）。
		return false;
	}
	// 如果新的策略是可进行替换的,但是我们本身是不可取消的,那么我们不能过渡
	if ((NewGroup == ELyraAbilityActivationGroup::Exclusive_Replaceable) && !CanBeCanceled())
	{
		// This ability can't become replaceable if it can't be canceled.
		// 如果这种能力无法被取消，那么它就无法被替代。
		return false;
	}

	return true;
}

```
``` cpp

bool ULyraGameplayAbility::ChangeActivationGroup(ELyraAbilityActivationGroup NewGroup)
{
	// GA的实例化策略
	ENSURE_ABILITY_IS_INSTANTIATED_OR_RETURN(ChangeActivationGroup, false);

	if (!CanChangeActivationGroup(NewGroup))
	{
		return false;
	}

	if (ActivationGroup != NewGroup)
	{
		ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponentFromActorInfo();
		check(LyraASC);

		// 移除旧组的计数
		LyraASC->RemoveAbilityFromActivationGroup(ActivationGroup, this);
		// 增加新组的计数
		LyraASC->AddAbilityToActivationGroup(NewGroup, this);

		// 将我们的策略组修改至新的
		ActivationGroup = NewGroup;
	}

	return true;
}

```


### 通过能力组互斥取消能力
``` cpp
void ULyraAbilitySystemComponent::CancelActivationGroupAbilities(ELyraAbilityActivationGroup Group, ULyraGameplayAbility* IgnoreLyraAbility, bool bReplicateCancelAbility)
{
	auto ShouldCancelFunc = [this, Group, IgnoreLyraAbility](const ULyraGameplayAbility* LyraAbility, FGameplayAbilitySpecHandle Handle)
	{
		return ((LyraAbility->GetActivationGroup() == Group) && (LyraAbility != IgnoreLyraAbility));
	};

	CancelAbilitiesByFunc(ShouldCancelFunc, bReplicateCancelAbility);
}
```
``` cpp
void ULyraAbilitySystemComponent::CancelAbilitiesByFunc(TShouldCancelAbilityFunc ShouldCancelFunc, bool bReplicateCancelAbility)
{
	ABILITYLIST_SCOPE_LOCK();
	for (const FGameplayAbilitySpec& AbilitySpec : ActivatableAbilities.Items)
	{
		// 该能力是非激活状态 可以跳过
		if (!AbilitySpec.IsActive())
		{
			continue;
		}

		// 没有授予的能力 跳过
		ULyraGameplayAbility* LyraAbilityCDO = Cast<ULyraGameplayAbility>(AbilitySpec.Ability);
		if (!LyraAbilityCDO)
		{
			UE_LOG(LogLyraAbilitySystem, Error, TEXT("CancelAbilitiesByFunc: Non-LyraGameplayAbility %s was Granted to ASC. Skipping."), *AbilitySpec.Ability.GetName());
			continue;
		}

		// 过期的能力非实例化策略 可以跳过
PRAGMA_DISABLE_DEPRECATION_WARNINGS
		ensureMsgf(AbilitySpec.Ability->GetInstancingPolicy() != EGameplayAbilityInstancingPolicy::NonInstanced, TEXT("CancelAbilitiesByFunc: All Abilities should be Instanced (NonInstanced is being deprecated due to usability issues)."));
PRAGMA_ENABLE_DEPRECATION_WARNINGS
			
		// Cancel all the spawned instances.
		// 取消所有生成的实例。
		TArray<UGameplayAbility*> Instances = AbilitySpec.GetAbilityInstances();
		
		for (UGameplayAbility* AbilityInstance : Instances)
		{
			ULyraGameplayAbility* LyraAbilityInstance = CastChecked<ULyraGameplayAbility>(AbilityInstance);

			// 根据传入的lambda是否需要取消
			if (ShouldCancelFunc(LyraAbilityInstance, AbilitySpec.Handle))
			{
				// 是否可以取消
				if (LyraAbilityInstance->CanBeCanceled())
				{
					/** 销毁按执行次序生成的实例化能力。每个角色的实例化能力应“重置”。任何处于激活状态的能力状态任务都会接收到“能力状态中断”事件。非实例化能力——我们该怎么办？*/
					LyraAbilityInstance->CancelAbility(AbilitySpec.Handle, AbilityActorInfo.Get(), LyraAbilityInstance->GetCurrentActivationInfo(), bReplicateCancelAbility);
				}
				else
				{
					// 不可被取消,但是需要取消,报错
					UE_LOG(LogLyraAbilitySystem, Error, TEXT("CancelAbilitiesByFunc: Can't cancel ability [%s] because CanBeCanceled is false."), *LyraAbilityInstance->GetName());
				}
			}
		}
	}
}

```
## 能力的消耗
``` cpp
	/** 检查费用。如果能够支付该能力的费用，则返回 true；否则返回 false */
	UE_API virtual bool CheckCost(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo,
		OUT FGameplayTagContainer* OptionalRelevantTags = nullptr) const override;
	
	/** 将该能力的消耗值施加于目标对象上 */
	UE_API virtual void ApplyCost(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo,
		const FGameplayAbilityActivationInfo ActivationInfo) const override;

```
``` cpp

bool ULyraGameplayAbility::CheckCost(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, OUT FGameplayTagContainer* OptionalRelevantTags) const
{
	if (!Super::CheckCost(Handle, ActorInfo, OptionalRelevantTags) || !ActorInfo)
	{
		return false;
	}

	// Verify we can afford any additional costs
	// 确认我们能够承担任何额外的费用
	for (const TObjectPtr<ULyraAbilityCost>& AdditionalCost : AdditionalCosts)
	{
		if (AdditionalCost != nullptr)
		{
			if (!AdditionalCost->CheckCost(this, Handle, ActorInfo, /*inout*/ OptionalRelevantTags))
			{
				return false;
			}
		}
	}

	return true;
}
```

``` cpp


void ULyraGameplayAbility::ApplyCost(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo) const
{
	Super::ApplyCost(Handle, ActorInfo, ActivationInfo);

	check(ActorInfo);

	// Used to determine if the ability actually hit a target (as some costs are only spent on successful attempts)
	// 用于判断该能力是否确实击中了目标（因为某些费用仅在成功攻击时才会消耗）
	auto DetermineIfAbilityHitTarget = [&]()
	{
		if (ActorInfo->IsNetAuthority())
		{
			if (ULyraAbilitySystemComponent* ASC = Cast<ULyraAbilitySystemComponent>(ActorInfo->AbilitySystemComponent.Get()))
			{
				FGameplayAbilityTargetDataHandle TargetData;
				ASC->GetAbilityTargetData(Handle, ActivationInfo, TargetData);
				for (int32 TargetDataIdx = 0; TargetDataIdx < TargetData.Data.Num(); ++TargetDataIdx)
				{
					if (UAbilitySystemBlueprintLibrary::TargetDataHasHitResult(TargetData, TargetDataIdx))
					{
						return true;
					}
				}
			}
		}

		return false;
	};

	// Pay any additional costs
	// 支付任何额外费用
	// 是否集中目标
	bool bAbilityHitTarget = false;
	// 是否已经执行过命中判定了
	bool bHasDeterminedIfAbilityHitTarget = false;
	for (const TObjectPtr<ULyraAbilityCost>& AdditionalCost : AdditionalCosts)
	{
		if (AdditionalCost != nullptr)
		{
			// 是否是只在命中时产生消耗
			if (AdditionalCost->ShouldOnlyApplyCostOnHit())
			{
				// 如果没有执行过命中判定,那么就在第一次的时候执行一次
				// 这样写阅读起来很奇怪 可能时考虑到只在这种情况下需要判定一次执行 不应当这样写
				if (!bHasDeterminedIfAbilityHitTarget)
				{
					bAbilityHitTarget = DetermineIfAbilityHitTarget();
					// 判定过了 后续不需要再判定了
					bHasDeterminedIfAbilityHitTarget = true;
				}
				// 根据判定结果 执行是否需要进行消耗.
				if (!bAbilityHitTarget)
				{
					continue;
				}
			}
			
			// 施加消耗
			AdditionalCost->ApplyCost(this, Handle, ActorInfo, ActivationInfo);
		}
	}
}
```
## 能力相机
``` cpp
	// Current camera mode set by the ability.
	// 当前的相机模式由该功能设定。
	TSubclassOf<ULyraCameraMode> ActiveCameraMode;

```
``` cpp
void ULyraGameplayAbility::SetCameraMode(TSubclassOf<ULyraCameraMode> CameraMode)
{
	ENSURE_ABILITY_IS_INSTANTIATED_OR_RETURN(SetCameraMode, );

	if (ULyraHeroComponent* HeroComponent = GetHeroComponentFromActorInfo())
	{
		HeroComponent->SetAbilityCameraMode(CameraMode, CurrentSpecHandle);
		ActiveCameraMode = CameraMode;
	}
}

void ULyraGameplayAbility::ClearCameraMode()
{
	ENSURE_ABILITY_IS_INSTANTIATED_OR_RETURN(ClearCameraMode, );

	if (ActiveCameraMode)
	{
		if (ULyraHeroComponent* HeroComponent = GetHeroComponentFromActorInfo())
		{
			HeroComponent->ClearAbilityCameraMode(CurrentSpecHandle);
		}

		ActiveCameraMode = nullptr;
	}
}

```
## 用于作弊时施加动态标签
``` cpp

	// Uses a gameplay effect to add the specified dynamic granted tag.
	// 使用游戏玩法效果来添加指定的动态授予标签。
	UE_API void AddDynamicTagGameplayEffect(const FGameplayTag& Tag);

	// Removes all active instances of the gameplay effect that was used to add the specified dynamic granted tag.
	// 清除所有使用指定动态授予标签来添加游戏效果时所激活的实例。
	UE_API void RemoveDynamicTagGameplayEffect(const FGameplayTag& Tag);
```
``` cpp
void ULyraAbilitySystemComponent::AddDynamicTagGameplayEffect(const FGameplayTag& Tag)
{
	// 这里需要读取我们全局使用的默认GE类别
	const TSubclassOf<UGameplayEffect> DynamicTagGE = ULyraAssetManager::GetSubclass(ULyraGameData::Get().DynamicTagGameplayEffect);
	
	if (!DynamicTagGE)
	{
		UE_LOG(LogLyraAbilitySystem, Warning, TEXT("AddDynamicTagGameplayEffect: Unable to find DynamicTagGameplayEffect [%s]."),
			*ULyraGameData::Get().DynamicTagGameplayEffect.GetAssetName());
		return;
	}
	/** 获取一个可直接应用于其他对象的输出游戏效果规格。*/
	const FGameplayEffectSpecHandle SpecHandle = MakeOutgoingSpec(DynamicTagGE, 1.0f, MakeEffectContext());
	FGameplayEffectSpec* Spec = SpecHandle.Data.Get();

	if (!Spec)
	{
		UE_LOG(LogLyraAbilitySystem, Warning, TEXT("AddDynamicTagGameplayEffect: Unable to make outgoing spec for [%s]."), *GetNameSafe(DynamicTagGE));
		return;
	}
	/** DynamicGrantedTags 所授予的、且并非源自 UGameplayEffect 定义的标签。这些标签会进行复制。*/
	Spec->DynamicGrantedTags.AddTag(Tag);
	
	/** 将之前创建的游戏玩法效果规格应用于此组件 */
	ApplyGameplayEffectSpecToSelf(*Spec);
}

void ULyraAbilitySystemComponent::RemoveDynamicTagGameplayEffect(const FGameplayTag& Tag)
{
	const TSubclassOf<UGameplayEffect> DynamicTagGE = ULyraAssetManager::GetSubclass(ULyraGameData::Get().DynamicTagGameplayEffect);
	if (!DynamicTagGE)
	{
		UE_LOG(LogLyraAbilitySystem, Warning, TEXT("RemoveDynamicTagGameplayEffect: Unable to find gameplay effect [%s]."), *ULyraGameData::Get().DynamicTagGameplayEffect.GetAssetName());
		return;
	}
	/** 创建一个效果查询，该查询将根据给定的标签与活跃游戏效果所属标签之间的共同标签情况进行匹配 */
	FGameplayEffectQuery Query = FGameplayEffectQuery::MakeQuery_MatchAnyOwningTags(FGameplayTagContainer(Tag));
	Query.EffectDefinition = DynamicTagGE;

	RemoveActiveEffects(Query);
}

```
## 总结
