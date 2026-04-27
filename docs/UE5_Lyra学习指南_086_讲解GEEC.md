# UE5_Lyra学习指南_086_讲解GEEC

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_086\_讲解GEEC](#ue5_lyra学习指南_086_讲解geec)
	- [概述](#概述)
	- [ULyraHealExecution](#ulyrahealexecution)
	- [ULyraDamageExecution](#ulyradamageexecution)
	- [总结](#总结)



## 概述
如代码所示即可.
## ULyraHealExecution
``` cpp
/**
 * ULyraHealExecution
 *
 *	Execution used by gameplay effects to apply healing to the health attributes.
 * 游戏玩法效果所使用的执行机制，用于对生命值属性进行治疗。
 *	
 */
UCLASS()
class ULyraHealExecution : public UGameplayEffectExecutionCalculation
{
	GENERATED_BODY()

public:

	ULyraHealExecution();

protected:

	virtual void Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const override;
};

```
``` cpp
struct FHealStatics
{
	/** 定义游戏玩法属性捕获选项的结构，用于游戏效果的设定 */
	FGameplayEffectAttributeCaptureDefinition BaseHealDef;

	FHealStatics()
	{
		BaseHealDef = FGameplayEffectAttributeCaptureDefinition(ULyraCombatSet::GetBaseHealAttribute(), EGameplayEffectAttributeCaptureSource::Source, true);
	}
};

static FHealStatics& HealStatics()
{
	static FHealStatics Statics;
	return Statics;
}


ULyraHealExecution::ULyraHealExecution()
{
	/** 用于记录与计算相关的属性 */
	RelevantAttributesToCapture.Add(HealStatics().BaseHealDef);
}

void ULyraHealExecution::Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
#if WITH_SERVER_CODE
	const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();

	const FGameplayTagContainer* SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
	const FGameplayTagContainer* TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

	FAggregatorEvaluateParameters EvaluateParameters;
	EvaluateParameters.SourceTags = SourceTags;
	EvaluateParameters.TargetTags = TargetTags;

	float BaseHeal = 0.0f;

	/**
	 * 根据指定参数计算所获取属性的大小。若游戏设定中该属性没有有效的获取方式，则可能会失败。
	 * @参数 InCaptureDef：需尝试计算其数值大小的属性定义
	 * @参数 InEvalParams：用于评估该属性的参数
	 * @参数 OutMagnitude：[输出] 计算出的数值大小*
	 * @返回值：如果成功计算出大小，则返回 True，否则返回 False
	 * 
	 */
	ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(HealStatics().BaseHealDef, EvaluateParameters, BaseHeal);

	const float HealingDone = FMath::Max(0.0f, BaseHeal);

	if (HealingDone > 0.0f)
	{
		// Apply a healing modifier, this gets turned into + health on the target
		// 应用治疗效果增强值，这会转化为对目标的 + 生命值加成效果。
		OutExecutionOutput.AddOutputModifier(FGameplayModifierEvaluatedData(ULyraHealthSet::GetHealingAttribute(), EGameplayModOp::Additive, HealingDone));
	}
#endif // #if WITH_SERVER_CODE
}



```

## ULyraDamageExecution
``` cpp

/**
 * ULyraDamageExecution
 *
 *	Execution used by gameplay effects to apply damage to the health attributes.
 * 游戏玩法效果所使用的执行机制，用于对生命值属性造成伤害。
 *
 *	
 */
UCLASS()
class ULyraDamageExecution : public UGameplayEffectExecutionCalculation
{
	GENERATED_BODY()

public:

	ULyraDamageExecution();

protected:

	virtual void Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const override;
};


```

``` cpp

struct FDamageStatics
{
	FGameplayEffectAttributeCaptureDefinition BaseDamageDef;

	FDamageStatics()
	{
		// 注意这里捕获的是自身的基础攻击力,我们目前设置为0.如果设置为10,那么手枪的伤害18,最终造成的伤害基数就是28.在这里数值的基础上再计算其他的折减系数
		BaseDamageDef = FGameplayEffectAttributeCaptureDefinition(ULyraCombatSet::GetBaseDamageAttribute(),
			EGameplayEffectAttributeCaptureSource::Source,
			true);
	}
};

static FDamageStatics& DamageStatics()
{
	static FDamageStatics Statics;
	return Statics;
}


ULyraDamageExecution::ULyraDamageExecution()
{
	/** 与计算相关的需要捕获的属性 */
	RelevantAttributesToCapture.Add(DamageStatics().BaseDamageDef);
}

void ULyraDamageExecution::Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
#if WITH_SERVER_CODE
	const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();
	// 获取我们到自定义的GE上下文
	FLyraGameplayEffectContext* TypedContext = FLyraGameplayEffectContext::ExtractEffectContext(Spec.GetContext());
	check(TypedContext);

	// 自身的Tag
	const FGameplayTagContainer* SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
	// 目标的Tag
	const FGameplayTagContainer* TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

	/** 用于聚合评估的数据，该数据由调用方/游戏代码传递过来 */
	FAggregatorEvaluateParameters EvaluateParameters;
	
	EvaluateParameters.SourceTags = SourceTags;
	EvaluateParameters.TargetTags = TargetTags;

	float BaseDamage = 0.0f;
	// 获取捕获的基础数值 例如手枪就是 0+18 =18
	ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(DamageStatics().BaseDamageDef, EvaluateParameters, BaseDamage);

	/** 返回与此效果关联的物理执行者 */
	const AActor* EffectCauser = TypedContext->GetEffectCauser();
	// 获取到命中的结果 可能为空  需要在前面的逻辑 如蓝图中 把命中的结果传递给上下文 这里才能收到
	const FHitResult* HitActorResult = TypedContext->GetHitResult();

	AActor* HitActor = nullptr;
	FVector ImpactLocation = FVector::ZeroVector;
	FVector ImpactNormal = FVector::ZeroVector;
	FVector StartTrace = FVector::ZeroVector;
	FVector EndTrace = FVector::ZeroVector;

	// Calculation of hit actor, surface, zone, and distance all rely on whether the calculation has a hit result or not.
	// Effects just being added directly w/o having been targeted will always come in without a hit result, which must default
	// to some fallback information.

	// 对击中角色、表面、区域以及距离的计算都取决于计算过程中是否产生了击中结果。
	// 未被目标击中的效果直接叠加进来时，通常不会产生击中结果，此时必须默认采用一些备用信息。
	if (HitActorResult)
	{
		const FHitResult& CurHitResult = *HitActorResult;
		HitActor = CurHitResult.HitObjectHandle.FetchActor();
		if (HitActor)
		{
			ImpactLocation = CurHitResult.ImpactPoint;
			ImpactNormal = CurHitResult.ImpactNormal;
			StartTrace = CurHitResult.TraceStart;
			EndTrace = CurHitResult.TraceEnd;
		}
	}

	// Handle case of no hit result or hit result not actually returning an actor
	// 处理未命中结果或命中结果实际上并未返回角色对象的情况
	UAbilitySystemComponent* TargetAbilitySystemComponent = ExecutionParams.GetTargetAbilitySystemComponent();
	if (!HitActor)
	{
		HitActor = TargetAbilitySystemComponent ? TargetAbilitySystemComponent->GetAvatarActor_Direct() : nullptr;
		if (HitActor)
		{
			ImpactLocation = HitActor->GetActorLocation();
		}
	}

	// Apply rules for team damage/self damage/etc...
	// 应用团队伤害/自身伤害等规则……
	float DamageInteractionAllowedMultiplier = 0.0f;
	if (HitActor)
	{
		ULyraTeamSubsystem* TeamSubsystem = HitActor->GetWorld()->GetSubsystem<ULyraTeamSubsystem>();
		if (ensure(TeamSubsystem))
		{
			// 友伤系数
			DamageInteractionAllowedMultiplier = TeamSubsystem->CanCauseDamage(EffectCauser, HitActor) ? 1.0 : 0.0;
		}
	}

	// Determine distance
	// 确定距离

	double Distance = WORLD_MAX;

	if (TypedContext->HasOrigin())
	{
		/** 返回起始点，若 HasOrigin 为 false 则该点可能无效 */
		
		Distance = FVector::Dist(TypedContext->GetOrigin(), ImpactLocation);
	}
	else if (EffectCauser)
	{
		Distance = FVector::Dist(EffectCauser->GetActorLocation(), ImpactLocation);
	}
	else
	{
		UE_LOG(LogLyraAbilitySystem, Error, TEXT("Damage Calculation cannot deduce a source location for damage coming from %s; Falling back to WORLD_MAX dist!"), *GetPathNameSafe(Spec.Def));
	}

	// Apply ability source modifiers
	// 应用能力来源的修正值
	float PhysicalMaterialAttenuation = 1.0f;
	float DistanceAttenuation = 1.0f;
	if (const ILyraAbilitySourceInterface* AbilitySource = TypedContext->GetAbilitySource())
	{	// 命中的物理材质 是PhysMat_Player 还是PhysMat_Player_WeakSpot?
		if (const UPhysicalMaterial* PhysMat = TypedContext->GetPhysicalMaterial())
		{
			// 读取武器上面的伤害系数
			// 在ULyraRangedWeaponInstance实现 需要查看B_WeaponInstance_Pistol
			// 可以看到手枪的弱点命中是2.0
			// 步枪是1.5
			// 霰弹枪是1.75
			PhysicalMaterialAttenuation = AbilitySource->GetPhysicalMaterialAttenuation(PhysMat, SourceTags, TargetTags);
		}
		// 读取武器上面的距离衰减系数
		DistanceAttenuation = AbilitySource->GetDistanceAttenuation(Distance, SourceTags, TargetTags);
	}
	// 距离衰减系数应该大于0
	DistanceAttenuation = FMath::Max(DistanceAttenuation, 0.0f);

	// Clamping is done when damage is converted to -health
	// 当伤害转化为“生命值减少”时，就会进行限制操作。
	// 基础伤害值经过折减后 应当不小于0
	const float DamageDone = FMath::Max(BaseDamage * DistanceAttenuation * PhysicalMaterialAttenuation * DamageInteractionAllowedMultiplier, 0.0f);

	if (DamageDone > 0.0f)
	{
		// Apply a damage modifier, this gets turned into - health on the target
		// 应用伤害修正值，这会转化为对目标的 - 生命值 影响。
		// 注意这里修改的不是CombatSet里面的BaseDamage 而是HealthSet里面的Damage!!!!
		OutExecutionOutput.AddOutputModifier(FGameplayModifierEvaluatedData(ULyraHealthSet::GetDamageAttribute(), EGameplayModOp::Additive, DamageDone));
	}
#endif // #if WITH_SERVER_CODE
}



```
## 总结
注意区分Modifiers和Executions
``` cpp
	// ------------------------------------------------------
	//	Modifiers
	//		These will modify the base value of attributes
	// ------------------------------------------------------
	
	
	// ------------------------------------------------------
	//	Executions
	//		This will run custom code to 'do stuff'
	// ------------------------------------------------------

```