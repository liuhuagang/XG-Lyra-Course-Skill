# UE5_Lyra学习指南_085_讲解属性集

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_085\_讲解属性集](#ue5_lyra学习指南_085_讲解属性集)
	- [概述](#概述)
	- [ULyraAttributeSet](#ulyraattributeset)
	- [ULyraCombatSet](#ulyracombatset)
	- [ULyraHealthSet](#ulyrahealthset)
		- [广播生命值变化](#广播生命值变化)
		- [代码](#代码)
	- [总结](#总结)



## 概述
本节主要创建属性值.
注意区分伤害计算流程中的几个伤害值属性.
## ULyraAttributeSet
``` cpp
/**
 * This macro defines a set of helper functions for accessing and initializing attributes.
 *
 * The following example of the macro:
 *		ATTRIBUTE_ACCESSORS(ULyraHealthSet, Health)
 * will create the following functions:
 *		static FGameplayAttribute GetHealthAttribute();
 *		float GetHealth() const;
 *		void SetHealth(float NewVal);
 *		void InitHealth(float NewVal);
 */
/**
 * 此宏定义了一组用于访问和初始化属性的辅助函数。以下这个宏的示例：
 *		ATTRIBUTE_ACCESSORS(ULyraHealthSet, Health)
 * 将会生成以下函数：
 *		静态函数 FGameplayAttribute GetHealthAttribute()；
 *		函数 float GetHealth() ；（该函数为常量函数）
 *		函数 void SetHealth(float NewVal) ；
 *		函数 void InitHealth(float NewVal) ；
 *		
 */
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

/** 
 * Delegate used to broadcast attribute events, some of these parameters may be null on clients: 
 * @param EffectInstigator	The original instigating actor for this event
 * @param EffectCauser		The physical actor that caused the change
 * @param EffectSpec		The full effect spec for this change
 * @param EffectMagnitude	The raw magnitude, this is before clamping
 * @param OldValue			The value of the attribute before it was changed
 * @param NewValue			The value after it was changed
*/
/**
 * 委托过去会广播属性事件，其中一些参数在客户端可能是空值：
 * @参数 作用源：此事件的原始触发者
 * @参数 作用原因：导致此变化的实际执行者
 * @参数 效果规格：此变化的完整效果规格
 * @参数 效果强度：原始强度值，即未进行限制之前
 * @参数 原始值：属性更改之前的状态值
 * @参数 新值：属性更改之后的状态值
 * 
 */
DECLARE_MULTICAST_DELEGATE_SixParams(FLyraAttributeEvent, AActor* /*EffectInstigator*/, AActor* /*EffectCauser*/, const FGameplayEffectSpec* /*EffectSpec*/, float /*EffectMagnitude*/, float /*OldValue*/, float /*NewValue*/);

/**
 * ULyraAttributeSet
 *
 *	Base attribute set class for the project.
 * 该项目的基础属性集类。
 *	
 */
UCLASS(MinimalAPI)
class ULyraAttributeSet : public UAttributeSet
{
	GENERATED_BODY()

public:

	UE_API ULyraAttributeSet();

	UE_API UWorld* GetWorld() const override;

	UE_API ULyraAbilitySystemComponent* GetLyraAbilitySystemComponent() const;
};


```
## ULyraCombatSet
``` cpp

/**
 * ULyraCombatSet
 *
 *  Class that defines attributes that are necessary for applying damage or healing.
 *	Attribute examples include: damage, healing, attack power, and shield penetrations.
 *
 * 定义了用于施加伤害或治疗所需属性的类。
 * 属性示例包括：伤害、治疗、攻击强度和护盾穿透力。
 *	
 */
UCLASS(BlueprintType)
class ULyraCombatSet : public ULyraAttributeSet
{
	GENERATED_BODY()

public:

	ULyraCombatSet();

	ATTRIBUTE_ACCESSORS(ULyraCombatSet, BaseDamage);
	ATTRIBUTE_ACCESSORS(ULyraCombatSet, BaseHeal);

protected:

	UFUNCTION()
	void OnRep_BaseDamage(const FGameplayAttributeData& OldValue);

	UFUNCTION()
	void OnRep_BaseHeal(const FGameplayAttributeData& OldValue);

private:

	// The base amount of damage to apply in the damage execution.
	// 在伤害执行过程中要施加的初始伤害值。
	UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_BaseDamage, Category = "Lyra|Combat", Meta = (AllowPrivateAccess = true))
	FGameplayAttributeData BaseDamage;

	// The base amount of healing to apply in the heal execution.
	// 在治疗执行过程中应用的初始治疗量。
	UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_BaseHeal, Category = "Lyra|Combat", Meta = (AllowPrivateAccess = true))
	FGameplayAttributeData BaseHeal;
};


```
``` cpp

ULyraCombatSet::ULyraCombatSet()
	: BaseDamage(10.0f)
	, BaseHeal(0.0f)
{
}

void ULyraCombatSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	DOREPLIFETIME_CONDITION_NOTIFY(ULyraCombatSet, BaseDamage, COND_OwnerOnly, REPNOTIFY_Always);
	DOREPLIFETIME_CONDITION_NOTIFY(ULyraCombatSet, BaseHeal, COND_OwnerOnly, REPNOTIFY_Always);
}

void ULyraCombatSet::OnRep_BaseDamage(const FGameplayAttributeData& OldValue)
{
	GAMEPLAYATTRIBUTE_REPNOTIFY(ULyraCombatSet, BaseDamage, OldValue);
}

void ULyraCombatSet::OnRep_BaseHeal(const FGameplayAttributeData& OldValue)
{
	GAMEPLAYATTRIBUTE_REPNOTIFY(ULyraCombatSet, BaseHeal, OldValue);
}

```

## ULyraHealthSet

### 广播生命值变化
修改前存储旧值
``` cpp

bool ULyraHealthSet::PreGameplayEffectExecute(FGameplayEffectModCallbackData &Data)
{
	if (!Super::PreGameplayEffectExecute(Data))
	{
		return false;
	}

	// Handle modifying incoming normal damage
	// 处理对进入式普通伤害的修改操作
	if (Data.EvaluatedData.Attribute == GetDamageAttribute())
	{
		if (Data.EvaluatedData.Magnitude > 0.0f)
		{
			const bool bIsDamageFromSelfDestruct = Data.EffectSpec.GetDynamicAssetTags().HasTagExact(TAG_Gameplay_DamageSelfDestruct);

			if (Data.Target.HasMatchingGameplayTag(TAG_Gameplay_DamageImmunity) && !bIsDamageFromSelfDestruct)
			{
				// Do not take away any health.
				// 不要损害任何健康。
				Data.EvaluatedData.Magnitude = 0.0f;
				return false;
			}

#if !UE_BUILD_SHIPPING
			// Check GodMode cheat, unlimited health is checked below
			// 检查“上帝模式”作弊功能，无限生命的情况将在下方进行检查
			if (Data.Target.HasMatchingGameplayTag(LyraGameplayTags::Cheat_GodMode) && !bIsDamageFromSelfDestruct)
			{
				// Do not take away any health.
				// 不要损害任何健康。
				Data.EvaluatedData.Magnitude = 0.0f;
				return false;
			}
#endif // #if !UE_BUILD_SHIPPING
		}
	}

	// Save the current health
	// 保存当前的生命值

	HealthBeforeAttributeChange = GetHealth();
	MaxHealthBeforeAttributeChange = GetMaxHealth();

	return true;
}

```

修改后传递新值和旧值
``` cpp


void ULyraHealthSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
	Super::PostGameplayEffectExecute(Data);

	const bool bIsDamageFromSelfDestruct = Data.EffectSpec.GetDynamicAssetTags().HasTagExact(TAG_Gameplay_DamageSelfDestruct);
	float MinimumHealth = 0.0f;

#if !UE_BUILD_SHIPPING
	// Godmode and unlimited health stop death unless it's a self destruct
	// 神态模式和无限生命可避免死亡，除非是自动毁灭的情况。
	if (!bIsDamageFromSelfDestruct &&
		(Data.Target.HasMatchingGameplayTag(LyraGameplayTags::Cheat_GodMode) || Data.Target.HasMatchingGameplayTag(LyraGameplayTags::Cheat_UnlimitedHealth) ))
	{
		MinimumHealth = 1.0f;
	}
#endif // #if !UE_BUILD_SHIPPING

	const FGameplayEffectContextHandle& EffectContext = Data.EffectSpec.GetEffectContext();
	AActor* Instigator = EffectContext.GetOriginalInstigator();
	AActor* Causer = EffectContext.GetEffectCauser();

	if (Data.EvaluatedData.Attribute == GetDamageAttribute())
	{
		// Send a standardized verb message that other systems can observe
		// 发送一条标准化的动词信息，以便其他系统能够进行观察
		if (Data.EvaluatedData.Magnitude > 0.0f)
		{
			FLyraVerbMessage Message;
			Message.Verb = TAG_Lyra_Damage_Message;
			Message.Instigator = Data.EffectSpec.GetEffectContext().GetEffectCauser();
			Message.InstigatorTags = *Data.EffectSpec.CapturedSourceTags.GetAggregatedTags();
			Message.Target = GetOwningActor();
			Message.TargetTags = *Data.EffectSpec.CapturedTargetTags.GetAggregatedTags();
			//@TODO: Fill out context tags, and any non-ability-system source/instigator tags
			//@TODO: Determine if it's an opposing team kill, self-own, team kill, etc...

			//@待办事项：填写情境标签，以及任何非能力系统来源/发起者标签
			//@待办事项：确定这是对方队伍的击杀、自身失误的击杀、团队击杀等情况。
			
			Message.Magnitude = Data.EvaluatedData.Magnitude;

			UGameplayMessageSubsystem& MessageSystem = UGameplayMessageSubsystem::Get(GetWorld());
			MessageSystem.BroadcastMessage(Message.Verb, Message);
		}

		// Convert into -Health and then clamp
		// 转换为“-健康”状态，然后进行限制/约束处理
		SetHealth(FMath::Clamp(GetHealth() - GetDamage(), MinimumHealth, GetMaxHealth()));
		SetDamage(0.0f);
	}
	else if (Data.EvaluatedData.Attribute == GetHealingAttribute())
	{
		// Convert into +Health and then clamo
		// Convert into +Health and then clamo

		SetHealth(FMath::Clamp(GetHealth() + GetHealing(), MinimumHealth, GetMaxHealth()));
		SetHealing(0.0f);
	}
	else if (Data.EvaluatedData.Attribute == GetHealthAttribute())
	{
		// Clamp and fall into out of health handling below
		// 进行限制处理并进入以下健康状况处理流程
		SetHealth(FMath::Clamp(GetHealth(), MinimumHealth, GetMaxHealth()));
	}
	else if (Data.EvaluatedData.Attribute == GetMaxHealthAttribute())
	{
		// TODO clamp current health?
		// TODO 对当前生命值进行限制？
		// Notify on any requested max health changes
		// 对于任何所要求的健康值最大值变化情况发出通知
		OnMaxHealthChanged.Broadcast(Instigator, Causer, &Data.EffectSpec, Data.EvaluatedData.Magnitude, MaxHealthBeforeAttributeChange, GetMaxHealth());
	}

	// If health has actually changed activate callbacks
	// 如果健康状况确实发生了变化，则激活回调函数
	if (GetHealth() != HealthBeforeAttributeChange)
	{
		OnHealthChanged.Broadcast(Instigator, Causer, &Data.EffectSpec, Data.EvaluatedData.Magnitude, HealthBeforeAttributeChange, GetHealth());
	}

	if ((GetHealth() <= 0.0f) && !bOutOfHealth)
	{
		OnOutOfHealth.Broadcast(Instigator, Causer, &Data.EffectSpec, Data.EvaluatedData.Magnitude, HealthBeforeAttributeChange, GetHealth());
	}

	// Check health again in case an event above changed it.
	// 再次检查一下健康状况，以防某个事件改变了其状态。
	bOutOfHealth = (GetHealth() <= 0.0f);
}

```

属性同步在客户端广播变化
``` cpp
void ULyraHealthSet::OnRep_Health(const FGameplayAttributeData& OldValue)
{
	GAMEPLAYATTRIBUTE_REPNOTIFY(ULyraHealthSet, Health, OldValue);

	// Call the change callback, but without an instigator
	// This could be changed to an explicit RPC in the future
	// These events on the client should not be changing attributes

	// 调用更改回调函数，但不指定触发者
	// 未来可能会将其改为明确的远程过程调用
	// 客户端上的这些事件不应更改属性
	

	const float CurrentHealth = GetHealth();
	const float EstimatedMagnitude = CurrentHealth - OldValue.GetCurrentValue();
	
	OnHealthChanged.Broadcast(nullptr, nullptr, nullptr, EstimatedMagnitude, OldValue.GetCurrentValue(), CurrentHealth);

	if (!bOutOfHealth && CurrentHealth <= 0.0f)
	{
		OnOutOfHealth.Broadcast(nullptr, nullptr, nullptr, EstimatedMagnitude, OldValue.GetCurrentValue(), CurrentHealth);
	}

	bOutOfHealth = (CurrentHealth <= 0.0f);
}

```
限制属性的修改范围
``` cpp

void ULyraHealthSet::PreAttributeBaseChange(const FGameplayAttribute& Attribute, float& NewValue) const
{
	Super::PreAttributeBaseChange(Attribute, NewValue);
	// 根据值类型 进行游戏玩法的限制
	ClampAttribute(Attribute, NewValue);
}

void ULyraHealthSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
	Super::PreAttributeChange(Attribute, NewValue);
	// 根据值类型 进行游戏玩法的限制
	ClampAttribute(Attribute, NewValue);
}

void ULyraHealthSet::PostAttributeChange(const FGameplayAttribute& Attribute, float OldValue, float NewValue)
{
	Super::PostAttributeChange(Attribute, OldValue, NewValue);

	if (Attribute == GetMaxHealthAttribute())
	{
		// Make sure current health is not greater than the new max health.
		// 确保当前的生命值不大于新的最大生命值。
		if (GetHealth() > NewValue)
		{
			ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent();
			check(LyraASC);

			LyraASC->ApplyModToAttribute(GetHealthAttribute(), EGameplayModOp::Override, NewValue);
		}
	}

	if (bOutOfHealth && (GetHealth() > 0.0f))
	{
		bOutOfHealth = false;
	}
}

void ULyraHealthSet::ClampAttribute(const FGameplayAttribute& Attribute, float& NewValue) const
{
	if (Attribute == GetHealthAttribute())
	{
		// Do not allow health to go negative or above max health.
		// 严禁使生命值降至负值或超过最大生命值。
		NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxHealth());
	}
	else if (Attribute == GetMaxHealthAttribute())
	{
		// Do not allow max health to drop below 1.
		// 严禁将最大生命值降至 1 以下。
		NewValue = FMath::Max(NewValue, 1.0f);
	}
}

```

### 代码
``` cpp

/**
 * ULyraHealthSet
 *
 *	Class that defines attributes that are necessary for taking damage.
 *	Attribute examples include: health, shields, and resistances.
 *
 * 定义承受伤害所需属性的类。
 * 属性示例包括：生命值、护盾和抗性。
 *	
 */
UCLASS(MinimalAPI, BlueprintType)
class ULyraHealthSet : public ULyraAttributeSet
{
	GENERATED_BODY()

public:

	UE_API ULyraHealthSet();

	ATTRIBUTE_ACCESSORS(ULyraHealthSet, Health);
	ATTRIBUTE_ACCESSORS(ULyraHealthSet, MaxHealth);
	ATTRIBUTE_ACCESSORS(ULyraHealthSet, Healing);
	ATTRIBUTE_ACCESSORS(ULyraHealthSet, Damage);

	// Delegate when health changes due to damage/healing, some information may be missing on the client
	// 当因受到伤害或恢复生命值而导致生命值变化时进行委托处理，客户端上可能会缺少一些信息。
	mutable FLyraAttributeEvent OnHealthChanged;

	// Delegate when max health changes
	// 当最大生命值发生变化时进行委托处理
	mutable FLyraAttributeEvent OnMaxHealthChanged;

	// Delegate to broadcast when the health attribute reaches zero
	// 当健康属性值降为零时，调用广播功能
	mutable FLyraAttributeEvent OnOutOfHealth;

protected:

	// 客户端接收生命值的变动 并传递到生命值组件去 由生命值组件进一步传递出去
	UFUNCTION()
	UE_API void OnRep_Health(const FGameplayAttributeData& OldValue);

	// 客户端接收最大生命值的变动 并传递到生命值组件去 由生命值组件进一步传递出去
	UFUNCTION()
	UE_API void OnRep_MaxHealth(const FGameplayAttributeData& OldValue);


	/**
	 * 在修改属性值之前调用。AttributeSet 可在此处进行其他修改操作。返回 true 表示继续执行，返回 false 则表示放弃此次修改。
	 * 请注意，此函数仅在“执行”过程中被调用。例如，对属性的“基础值”的修改。它不会在应用游戏效果（如持续 5 秒、增加 10 移动速度的增益效果）时被调用。
	 * 
	 */
	// 在GE施加前 存储旧值 并做好无敌或者免疫伤害的tag检定 从而修改实际造成的伤害值
	UE_API virtual bool PreGameplayEffectExecute(FGameplayEffectModCallbackData& Data) override;

	/**
	 * 在执行游戏效果后调用，用于修改某个属性的基础值。此后不能再进行任何修改。
	 * 注意，此函数仅在“执行”阶段被调用。例如，对某个属性“基础值”的修改。它不会在应用游戏效果（如持续 5 秒且增加 10 点移动速度的增益效果）时被调用。
	 * 
	 */
	// 在GE施加后 拿到GE造成的值和值类型 修改对应的属性 并广播出去
	UE_API virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data) override;
	/**
	 * 这是在对属性的基础值进行任何修改之前所发生的情况，此时存在属性聚合器。
	 * 此函数应执行限制操作（假设您希望在“预属性更改”阶段同时对基础值和最终值进行限制）
	 * 此函数不应调用与游戏玩法相关的事件或回调。请在“预属性更改”阶段（该阶段会在属性的实际最终值发生变化之前被调用）中进行这些操作。
	 *  
	 */
	UE_API virtual void PreAttributeBaseChange(const FGameplayAttribute& Attribute, float& NewValue) const override;
	/**
	 * 这是在对属性的基础值进行任何修改之前所发生的情况，此时存在属性聚合器。
	 * 此函数应执行限制操作（假设您希望在“预属性更改”阶段同时对基础值和最终值进行限制）
	 * 此函数不应调用与游戏玩法相关的事件或回调。请在“预属性更改”阶段（该阶段会在属性的实际最终值发生变化之前被调用）中进行这些操作。
	 * 
	 */
	UE_API virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;

	/** 在对任何属性进行修改后会自动调用此函数。*/
	UE_API virtual void PostAttributeChange(const FGameplayAttribute& Attribute, float OldValue, float NewValue) override;

	// 限制值
	UE_API void ClampAttribute(const FGameplayAttribute& Attribute, float& NewValue) const;

private:

	// The current health attribute.  The health will be capped by the max health attribute.  Health is hidden from modifiers so only executions can modify it.
	// 当前的生命值属性。生命值会受到最大生命值属性的限制。生命值不会被任何修饰符所影响，只有执行操作时才能对其进行修改。
	UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_Health, Category = "Lyra|Health", Meta = (HideFromModifiers, AllowPrivateAccess = true))
	FGameplayAttributeData Health;

	// The current max health attribute.  Max health is an attribute since gameplay effects can modify it.
	// 当前的最大生命值属性。最大生命值是一个属性，因为游戏过程中的某些因素可能会对其进行调整。
	UPROPERTY(BlueprintReadOnly, ReplicatedUsing = OnRep_MaxHealth, Category = "Lyra|Health", Meta = (AllowPrivateAccess = true))
	FGameplayAttributeData MaxHealth;

	// Used to track when the health reaches 0.
	// 用于记录健康值降至 0 的时间。
	bool bOutOfHealth;

	// Store the health before any changes
	// 在进行任何更改之前先保存健康状况
	float MaxHealthBeforeAttributeChange;
	float HealthBeforeAttributeChange;

	// -------------------------------------------------------------------
	//	Meta Attribute (please keep attributes that aren't 'stateful' below
	//  元属性（请将那些“非状态性”的属性保留在下方）
	// -------------------------------------------------------------------

	// Incoming healing. This is mapped directly to +Health
	// 正在施加治疗效果。此效果直接对应于“生命值+”的效果。
	UPROPERTY(BlueprintReadOnly, Category="Lyra|Health", Meta=(AllowPrivateAccess=true))
	FGameplayAttributeData Healing;

	// Incoming damage. This is mapped directly to -Health
	// 正在受到的伤害。此伤害值直接映射到“生命值”上。
	UPROPERTY(BlueprintReadOnly, Category="Lyra|Health", Meta=(HideFromModifiers, AllowPrivateAccess=true))
	FGameplayAttributeData Damage;
};

```



## 总结
本节主要介绍了属性集.注意这里有两个地方没有拓展展开.
1.造成伤害时去记录伤害的历史,从而得到助攻数据
2.死亡时生命值组件触发死亡.