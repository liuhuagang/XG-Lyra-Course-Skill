# UE5_Lyra学习指南_121_炸弹人玩法

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_121\_炸弹人玩法](#ue5_lyra学习指南_121_炸弹人玩法)
	- [概述](#概述)
	- [角色数据](#角色数据)
	- [相机模式](#相机模式)
	- [移动组件](#移动组件)
	- [属性集](#属性集)
	- [GE的UI描述](#ge的ui描述)
	- [死亡技能](#死亡技能)
	- [丢炸弹技能](#丢炸弹技能)
	- [炸弹基类](#炸弹基类)
		- [放置状态](#放置状态)
		- [燃烧状态](#燃烧状态)
		- [递归爆炸](#递归爆炸)
		- [四方向生成效果](#四方向生成效果)
		- [目标方向的重叠检测](#目标方向的重叠检测)
		- [摧毁恢复炸弹可放置数量](#摧毁恢复炸弹可放置数量)
		- [放置炸弹时避免卡住自己](#放置炸弹时避免卡住自己)
	- [标准炸弹](#标准炸弹)
	- [可拾取物](#可拾取物)
		- [属性同步修改样式](#属性同步修改样式)
		- [效果施加](#效果施加)
	- [关卡生成器](#关卡生成器)
	- [GameFeatureData](#gamefeaturedata)
	- [输入资产](#输入资产)
	- [Experience](#experience)
		- [HUD](#hud)
			- [拓展程序](#拓展程序)
			- [拓展点](#拓展点)
	- [队伍](#队伍)
	- [游戏积分](#游戏积分)
	- [阶段切换](#阶段切换)
	- [人物选择](#人物选择)
	- [拾取特效](#拾取特效)
	- [开场特效](#开场特效)
	- [总结](#总结)



## 概述
本节主要简单介绍一下俯视角下的炸弹人玩法构建.
## 角色数据
![P_PawnData](./Pictures/035Boom/P_PawnData.png)

## 相机模式
![P_CM](./Pictures/035Boom/P_CM.png)
``` cpp
/**
 * ULyraCameraMode_TopDownArenaCamera
 *
 *	A basic third person camera mode that looks down at a fixed arena.
 *	一种基础的第三人称视角模式，从上方俯瞰固定的竞技场。
 */
UCLASS(Abstract, Blueprintable)
class ULyraCameraMode_TopDownArenaCamera : public ULyraCameraMode
{
	GENERATED_BODY()

public:

	ULyraCameraMode_TopDownArenaCamera();

protected:

	//~ULyraCameraMode interface
	virtual void UpdateView(float DeltaTime) override;
	//~End of ULyraCameraMode interface

protected:
	UPROPERTY(EditDefaultsOnly, Category = "Third Person")
	float ArenaWidth;

	UPROPERTY(EditDefaultsOnly, Category = "Third Person")
	float ArenaHeight;

	UPROPERTY(EditDefaultsOnly, Category = "Third Person")
	FRotator DefaultPivotRotation;

	UPROPERTY(EditDefaultsOnly, Category = "Third Person")
	FRuntimeFloatCurve BoundsSizeToDistance;
};


```
## 移动组件
![P_MoveComp](./Pictures/035Boom/P_MoveComp.png)
``` cpp
UCLASS()
class UTopDownArenaMovementComponent : public ULyraCharacterMovementComponent
{
	GENERATED_BODY()

public:

	UTopDownArenaMovementComponent(const FObjectInitializer& ObjectInitializer);

	//~UMovementComponent interface
	virtual float GetMaxSpeed() const override;
	//~End of UMovementComponent interface

};


```
``` cpp
UTopDownArenaMovementComponent::UTopDownArenaMovementComponent(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
{
}

float UTopDownArenaMovementComponent::GetMaxSpeed() const
{
	if (UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(GetOwner()))
	{
		if (MovementMode == MOVE_Walking)
		{
			if (ASC->HasMatchingGameplayTag(TAG_Gameplay_MovementStopped))
			{
				return 0;
			}

			const float MaxSpeedFromAttribute = ASC->GetNumericAttribute(UTopDownArenaAttributeSet::GetMovementSpeedAttribute());
			if (MaxSpeedFromAttribute > 0.0f)
			{
				return MaxSpeedFromAttribute;
			}
		}
	}

	return Super::GetMaxSpeed();
}


```

## 属性集
![P_AbilitySet](./Pictures/035Boom/P_AbilitySet.png)
``` cpp
/**
 * UTopDownArenaAttributeSet
 *
 *	Class that defines attributes specific to the top-down arena gameplay mode.
 */
UCLASS(BlueprintType)
class UTopDownArenaAttributeSet : public ULyraAttributeSet
{
	GENERATED_BODY()

public:
	UTopDownArenaAttributeSet();

	ATTRIBUTE_ACCESSORS(ThisClass, BombsRemaining);
	ATTRIBUTE_ACCESSORS(ThisClass, BombCapacity);
	ATTRIBUTE_ACCESSORS(ThisClass, BombRange);
	ATTRIBUTE_ACCESSORS(ThisClass, MovementSpeed);

	//~UAttributeSet interface
	virtual void PreAttributeBaseChange(const FGameplayAttribute& Attribute, float& NewValue) const override;
	virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;
	//~End of UAttributeSet interface

protected:

	UFUNCTION()
	void OnRep_BombsRemaining(const FGameplayAttributeData& OldValue);

	UFUNCTION()
	void OnRep_BombCapacity(const FGameplayAttributeData& OldValue);

	UFUNCTION()
	void OnRep_BombRange(const FGameplayAttributeData& OldValue);
	
	UFUNCTION()
	void OnRep_MovementSpeed(const FGameplayAttributeData& OldValue);

	void ClampAttribute(const FGameplayAttribute& Attribute, float& NewValue) const;

private:
	// The number of bombs remaining
	// 剩下的炸弹数量
	UPROPERTY(BlueprintReadOnly, ReplicatedUsing=OnRep_BombsRemaining, Category="TopDownArenaGame", Meta=(AllowPrivateAccess=true))
	FGameplayAttributeData BombsRemaining;

	// The maximum number of bombs that can be placed at once
	// 炸弹最大容量
	UPROPERTY(BlueprintReadOnly, ReplicatedUsing=OnRep_BombCapacity, Category="TopDownArenaGame", Meta=(AllowPrivateAccess=true))
	FGameplayAttributeData BombCapacity;

	// The range/radius of bomb blasts
	// 炸弹的伤害范围
	UPROPERTY(BlueprintReadOnly, ReplicatedUsing=OnRep_BombRange, Category="TopDownArenaGame", Meta=(AllowPrivateAccess=true))
	FGameplayAttributeData BombRange;

	// The range/radius of bomb blasts
	// 移动速度
	UPROPERTY(BlueprintReadOnly, ReplicatedUsing=OnRep_MovementSpeed, Category="TopDownArenaGame", Meta=(AllowPrivateAccess=true))
	FGameplayAttributeData MovementSpeed;
};


```
``` cpp
UTopDownArenaAttributeSet::UTopDownArenaAttributeSet()
	: BombsRemaining(1.0f)
	, BombCapacity(1.0f)
	, BombRange(2.0f)
	, MovementSpeed(400.0f)
{
}

void UTopDownArenaAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	DOREPLIFETIME_CONDITION_NOTIFY(ThisClass, BombsRemaining, COND_None, REPNOTIFY_Always);
	DOREPLIFETIME_CONDITION_NOTIFY(ThisClass, BombCapacity, COND_None, REPNOTIFY_Always);
	DOREPLIFETIME_CONDITION_NOTIFY(ThisClass, BombRange, COND_None, REPNOTIFY_Always);
	DOREPLIFETIME_CONDITION_NOTIFY(ThisClass, MovementSpeed, COND_None, REPNOTIFY_Always);
}

void UTopDownArenaAttributeSet::OnRep_BombsRemaining(const FGameplayAttributeData& OldValue)
{
	GAMEPLAYATTRIBUTE_REPNOTIFY(ThisClass, BombsRemaining, OldValue);
}

void UTopDownArenaAttributeSet::OnRep_BombCapacity(const FGameplayAttributeData& OldValue)
{
	GAMEPLAYATTRIBUTE_REPNOTIFY(ThisClass, BombCapacity, OldValue);
}

void UTopDownArenaAttributeSet::OnRep_BombRange(const FGameplayAttributeData& OldValue)
{
	GAMEPLAYATTRIBUTE_REPNOTIFY(ThisClass, BombRange, OldValue);
}

void UTopDownArenaAttributeSet::OnRep_MovementSpeed(const FGameplayAttributeData& OldValue)
{
	GAMEPLAYATTRIBUTE_REPNOTIFY(ThisClass, MovementSpeed, OldValue);
}

void UTopDownArenaAttributeSet::PreAttributeBaseChange(const FGameplayAttribute& Attribute, float& NewValue) const
{
	Super::PreAttributeBaseChange(Attribute, NewValue);

	ClampAttribute(Attribute, NewValue);
}

void UTopDownArenaAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
	Super::PreAttributeChange(Attribute, NewValue);

	ClampAttribute(Attribute, NewValue);
}

void UTopDownArenaAttributeSet::ClampAttribute(const FGameplayAttribute& Attribute, float& NewValue) const
{
	if (Attribute == GetBombsRemainingAttribute())
	{
		NewValue = FMath::Clamp(NewValue, 0.0f, GetBombCapacity());
	}
	else if (Attribute == GetBombCapacityAttribute())
	{
		NewValue = FMath::Max(NewValue, 1.0f);
	}
	else if (Attribute == GetBombRangeAttribute())
	{
		NewValue = FMath::Max(NewValue, 1.0f);
	}
	else if (Attribute == GetMovementSpeedAttribute())
	{
		NewValue = FMath::Clamp(NewValue, 200.0f, 800.0f);
	}
}
```

## GE的UI描述
![P_GE_HealthUp](./Pictures/035Boom/P_GE_HealthUp.png)
![P_ReadGEUI](./Pictures/035Boom/P_ReadGEUI.png)
``` cpp
/**
 * UGameplayEffectUIData
 * Base class to provide game-specific data about how to describe a Gameplay Effect in the UI. Subclass with data to use in your game.
 * In Unreal Engine 5.3, this now derives from UGameplayEffectComponent so you can use it directly as a GameplayEffectComponent.
 */
UCLASS(Blueprintable, Abstract, EditInlineNew, CollapseCategories, MinimalAPI)
class UGameplayEffectUIData : public UGameplayEffectComponent
{
	GENERATED_BODY()
};


```
``` cpp
// Icon and display name for pickups in the top-down arena game
UCLASS(BlueprintType)
class UTopDownArenaPickupUIData : public UGameplayEffectUIData
{
	GENERATED_BODY()

public:

	// The full description of the pickup
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Data, meta=(MultiLine="true"))
	FText Description;

	// The short description of the pickup (displayed by the player name when picked up)
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Data, meta=(MultiLine="true"))
	FText ShortDescriptionForToast;
	
	// The icon material used to show the pickup in the world
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=Data)
	TObjectPtr<UTexture2D> IconTexture;

	// The pickup VFX override
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = Data)
	TObjectPtr<UNiagaraSystem> PickupVFX;

	// The pickup SFX override (if not set, a default will play)
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = Data)
	TObjectPtr<USoundBase> PickupSFX;
};

```
## 死亡技能
GA_ArenaHero_Death
![P_GA_ArenaHero_Death](./Pictures/035Boom/P_GA_ArenaHero_Death.png)

## 丢炸弹技能
GA_DropBomb
![P_GA_DropBomb](./Pictures/035Boom/P_GA_DropBomb.png)
GE_DecrementBombsRemaining
消耗炸弹数量可使用的
GE_IncrementBombsRemaining
提升炸弹数量可使用的
对齐到网格
![P_BoomGrid](./Pictures/035Boom/P_BoomGrid.png)

## 炸弹基类
炸弹有三种状态：
- 已放置
- 燃烧
- 爆炸（每次爆炸会波及相邻一格，直至达到最大爆炸半径）

### 放置状态
![P_BoomStatua1_Place](./Pictures/035Boom/P_BoomStatua1_Place.png)
### 燃烧状态
![P_BoomStatua2_Boom.png](./Pictures/035Boom/P_BoomStatua2_Boom.png)
### 递归爆炸
![P_BoomStatua3_Increment](./Pictures/035Boom/P_BoomStatua3_Increment.png)
### 四方向生成效果
![P_Boom_4Direction](./Pictures/035Boom/P_Boom_4Direction.png)
注意这里有一个bool的数组如果这个方向检测到阻拦了就不再生成这个方向的火球实例子了
### 目标方向的重叠检测
![P_DirectionDetect](./Pictures/035Boom/P_DirectionDetect.png)
![P_Boom_Detect4](./Pictures/035Boom/P_Boom_Detect4.png)
1.如果检测到静态网格体.一般来说就是堵住了.不需要生成黑色的火球.这是ok的
如果什么都没有检测到.说明是空的,需要添加一个黑色的火球用于调试,并且用于属性同步的数组在客户端生成效果
![P_Collision](./Pictures/035Boom/P_Collision.png)
2.如果检测到炸弹,就触发这个炸弹的爆炸即可
3.检测到可以破坏的物体,则生成可以拾取物品,有一定的随机概率
![P_SpawnBonus](./Pictures/035Boom/P_SpawnBonus.png)
4.如果检测到人物就造成伤害.

isblocked会返回给上级,从而知道是否需要触发这个方向的递归.
SpawnedFireball会传递给子类用于生成效果
![P_DirectionalRadius](./Pictures/035Boom/P_DirectionalRadius.png)
![P_OnRep_RadiusState](./Pictures/035Boom/P_OnRep_RadiusState.png)
![P_Bomb_Standard](./Pictures/035Boom/P_Bomb_Standard.png)

### 摧毁恢复炸弹可放置数量
![P_DestoryBoom](./Pictures/035Boom/P_DestoryBoom.png)
### 放置炸弹时避免卡住自己
![P_EndOverlap](./Pictures/035Boom/P_EndOverlap.png)
## 标准炸弹
![P_Boom_Standard](./Pictures/035Boom/P_Boom_Standard.png)
## 可拾取物
![P_Pickup](./Pictures/035Boom/P_Pickup.png)
### 属性同步修改样式
![P_ChangeEffect](./Pictures/035Boom/P_ChangeEffect.png)
![P_ChangeEffectFromRep](./Pictures/035Boom/P_ChangeEffectFromRep.png)
### 效果施加
![P_ApplyPickup](./Pictures/035Boom/P_ApplyPickup.png)

## 关卡生成器
![P_Generate](./Pictures/035Boom/P_Generate.png)
里面的参数适用于L_TopDown_LocalMultiplayer
而非L_TopDownArenaGym
![P_ShowGenerate](./Pictures/035Boom/P_ShowGenerate.png)
做好网格计算即可.
![P_Destruct](./Pictures/035Boom/P_Destruct.png)
## GameFeatureData
![P_Core](./Pictures/035Boom/P_Core.png)
主要是添加扫描路径
## 输入资产

![P_Input](./Pictures/035Boom/P_Input.png)

![P_InputData](./Pictures/035Boom/P_InputData.png)
## Experience
### HUD
这里的写法稍微有点绕
#### 拓展程序
![P_UIComp](./Pictures/035Boom/P_UIComp.png)
#### 拓展点

![P_HUDLayout](./Pictures/035Boom/P_HUDLayout.png)
![P_TopLayout](./Pictures/035Boom/P_TopLayout.png)
![P_TopWidget](./Pictures/035Boom/P_TopWidget.png)
``` cpp

void UUIExtensionPointWidget::OnAddOrRemoveExtension(EUIExtensionAction Action, const FUIExtensionRequest& Request)
{
	// 接收回调过来的拓展事件
	if (Action == EUIExtensionAction::Added)
	{
		// 从拓展程序里面获取到要实例化的控件
		UObject* Data = Request.Data;
		
		TSubclassOf<UUserWidget> WidgetClass(Cast<UClass>(Data));
		if (WidgetClass)
		{
			UUserWidget* Widget = CreateEntryInternal(WidgetClass);
			ExtensionMapping.Add(Request.ExtensionHandle, Widget);
		}
		else if (DataClasses.Num() > 0)
		{
			if (GetWidgetClassForData.IsBound())
			{
				// 从Data获取控件类型
				WidgetClass = GetWidgetClassForData.Execute(Data);

				// If the data is irrelevant they can just return no widget class.
				// 如果这些数据与主题无关，他们就可以直接不提供任何组件类。
				if (WidgetClass)
				{
					if (UUserWidget* Widget = CreateEntryInternal(WidgetClass))
					{
						ExtensionMapping.Add(Request.ExtensionHandle, Widget);
						ConfigureWidgetForData.ExecuteIfBound(Widget, Data);
					}
				}
			}
		}
	}
	else
	{
		// 如果不是添加事件 移除即可
		if (UUserWidget* Extension = ExtensionMapping.FindRef(Request.ExtensionHandle))
		{
			RemoveEntryInternal(Extension);
			ExtensionMapping.Remove(Request.ExtensionHandle);
		}
	}
}

```
拿到类
![P_GetWidgetClass](./Pictures/035Boom/P_GetWidgetClass.png)
拿到数据去配置
![P_GetWidgetData](./Pictures/035Boom/P_GetWidgetData.png)
转发给子项去绑定
![P_PlayTileList](./Pictures/035Boom/P_PlayTileList.png)
![P_TileToConfigure](./Pictures/035Boom/P_TileToConfigure.png)
![P_PowerUpStat](./Pictures/035Boom/P_PowerUpStat.png)
![P_BindAttribute](./Pictures/035Boom/P_BindAttribute.png)

数据是从这里过来的
![P_RegisterUserData](./Pictures/035Boom/P_RegisterUserData.png)
![P_PlayerUIComponent](./Pictures/035Boom/P_PlayerUIComponent.png)

## 队伍

![P_4Teams](./Pictures/035Boom/P_4Teams.png)
## 游戏积分
B_TopDownArena_GameComponent_Base
![P_Count](./Pictures/035Boom/P_Count.png)
## 阶段切换
![P_Phase](./Pictures/035Boom/P_Phase.png)
## 人物选择
![P_PickCharacter](./Pictures/035Boom/P_PickCharacter.png)
## 拾取特效
![P_GCNL_Pickup](./Pictures/035Boom/P_GCNL_Pickup.png)
![P_GE_BasePickup](./Pictures/035Boom/P_GE_BasePickup.png)
## 开场特效
![P_GetReady](./Pictures/035Boom/P_GetReady.png)
![P_Warmup](./Pictures/035Boom/P_Warmup.png)


## 总结
本节简要的提及了关于俯角炸弹人的玩法设计.主要是以蓝图为主.
其中有几个稍微难一点的东西:
1.UUIExtensionPointWidget::OnAddOrRemoveExtension如何绑定,如何发送数据
2.用于监听Attribute的Task.这个是GAS写的.
3.GE的UGameplayEffectUIData
注意有些特性是较新的引擎版本才有!!!