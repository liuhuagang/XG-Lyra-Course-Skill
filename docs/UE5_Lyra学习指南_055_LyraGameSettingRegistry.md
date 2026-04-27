# UE5_Lyra学习指南_055_LyraGameSettingRegistry

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_055\_LyraGameSettingRegistry](#ue5_lyra学习指南_055_lyragamesettingregistry)
	- [概述](#概述)
	- [代码](#代码)
		- [LyraGameSettingRegistry](#lyragamesettingregistry)
		- [GameSettingRegistry](#gamesettingregistry)
	- [获取游戏设置注册器](#获取游戏设置注册器)
		- [关于Outer是否会影响NewObject的生命周期](#关于outer是否会影响newobject的生命周期)
		- [NewObject](#newobject)
		- [FindObeject](#findobeject)
		- [测试Outer的生命周期代码](#测试outer的生命周期代码)
	- [真正获取初始化游戏设置注册器](#真正获取初始化游戏设置注册器)
	- [初始化游戏设置](#初始化游戏设置)
		- [初始化流程](#初始化流程)
		- [创建游戏设置集合](#创建游戏设置集合)
		- [Get/Set方法](#getset方法)
	- [数据源绑定](#数据源绑定)
		- [注册路径](#注册路径)
		- [解算路径](#解算路径)
	- [注册游戏设置](#注册游戏设置)
	- [总结](#总结)



## 概述
LyraGameSettingRegistry非常重要,它决定我们要实例化那些具体的游戏设置!
## 代码
先直接贴出LyraGameSettingRegistry和UGameSettingRegistry的代码
### LyraGameSettingRegistry
``` cpp

/**
 * 游戏属性注册
 */
UCLASS()
class ULyraGameSettingRegistry : public UGameSettingRegistry
{
	GENERATED_BODY()

public:
	// 构造函数 无作用
	ULyraGameSettingRegistry();

	// 这个写法是错误的 因为会有GC问题 Outer并不能防止GC 当然也有可能它就是这样动态设计的
	// 实际上由控件蓝图Screen来实例化注册器并持有它
	static ULyraGameSettingRegistry* Get(ULyraLocalPlayer* InLocalPlayer);

	// 通知我们得设置类 应用保存即可
	virtual void SaveChanges() override;

protected:
	// 重写父类的方法 实际初始化
	virtual void OnInitialize(ULocalPlayer* InLocalPlayer) override;
	// 重写父类的方法 必须要等待共享设置加载完成之后才行
	virtual bool IsFinishedInitializing() const override;

	// 创建Video页面
	UGameSettingCollection* InitializeVideoSettings(ULyraLocalPlayer* InLocalPlayer);
	// 在Video页面填充帧率限制选项
	void InitializeVideoSettings_FrameRates(UGameSettingCollection* Screen, ULyraLocalPlayer* InLocalPlayer);
	// 添加关于性能的游戏设置集合
	void AddPerformanceStatPage(UGameSettingCollection* Screen, ULyraLocalPlayer* InLocalPlayer);

	// 初始化音频设置
	UGameSettingCollection* InitializeAudioSettings(ULyraLocalPlayer* InLocalPlayer);

	// 初始化游戏玩法设置
	UGameSettingCollection* InitializeGameplaySettings(ULyraLocalPlayer* InLocalPlayer);

	// 初始化键数设置
	UGameSettingCollection* InitializeMouseAndKeyboardSettings(ULyraLocalPlayer* InLocalPlayer);

	// 初始化手柄设置
	UGameSettingCollection* InitializeGamepadSettings(ULyraLocalPlayer* InLocalPlayer);

	// 持有的视频设置集合
	UPROPERTY()
	TObjectPtr<UGameSettingCollection> VideoSettings;

	// 持有的音频设置集合
	UPROPERTY()
	TObjectPtr<UGameSettingCollection> AudioSettings;

	// 持有的游戏玩法设置集合
	UPROPERTY()
	TObjectPtr<UGameSettingCollection> GameplaySettings;

	// 持有的键鼠设置集合
	UPROPERTY()
	TObjectPtr<UGameSettingCollection> MouseAndKeyboardSettings;

	// 持有的手柄设置集合
	UPROPERTY()
	TObjectPtr<UGameSettingCollection> GamepadSettings;
};


```
### GameSettingRegistry
``` cpp

/**
 * 游戏设置注册器 用来连接具体的游戏设置和UI 承上启下
 */
UCLASS(MinimalAPI, Abstract, BlueprintType)
class UGameSettingRegistry : public UObject
{
	GENERATED_BODY()

public:
	// 定义 游戏设置发生改变的代理
	DECLARE_EVENT_TwoParams(UGameSettingRegistry, FOnSettingChanged, UGameSetting*, EGameSettingChangeReason);

	// 定义 设置编辑条件发生改变的代理
	DECLARE_EVENT_OneParam(UGameSettingRegistry, FOnSettingEditConditionChanged, UGameSetting*);

	// 游戏设置发生改变的代理
	FOnSettingChanged OnSettingChangedEvent;
	// 设置编辑条件发生改变的代理
	FOnSettingEditConditionChanged OnSettingEditConditionChangedEvent;

	// 设置被命名的行动事件
	DECLARE_EVENT_TwoParams(UGameSettingRegistry, FOnSettingNamedActionEvent, UGameSetting* /*Setting*/, FGameplayTag /*GameSettings_Action_Tag*/);
	FOnSettingNamedActionEvent OnSettingNamedActionEvent;

	/** Navigate to the child settings of the provided setting. */
	/** 跳转至所提供的设置的子设置页面。*/
	DECLARE_EVENT_OneParam(UGameSettingRegistry, FOnExecuteNavigation, UGameSetting* /*Setting*/);
	FOnExecuteNavigation OnExecuteNavigationEvent;

public:
	// 构造函数 无作用
	UE_API UGameSettingRegistry();

	// 初始化 设置本地玩家类
	// 走这里初始化的ULyraSettingScreen::CreateRegistry()
	UE_API void Initialize(ULocalPlayer* InLocalPlayer);

	// 重新生成
	UE_API virtual void Regenerate();

	// 是否完成初始
	UE_API virtual bool IsFinishedInitializing() const;

	// 保存变化 由UGameSettingScreen::ApplyChanges调用
	UE_API virtual void SaveChanges();

	// 过滤游戏设置
	UE_API void GetSettingsForFilter(const FGameSettingFilterState& FilterState, TArray<UGameSetting*>& InOutSettings);

	// 根据名字找到对应的游戏设置
	UE_API UGameSetting* FindSettingByDevName(const FName& SettingDevName);

	// 带有检测的获取游戏设置
	template<typename T = UGameSetting>
	T* FindSettingByDevNameChecked(const FName& SettingDevName)
	{
		T* Setting = Cast<T>(FindSettingByDevName(SettingDevName));
		check(Setting);
		return Setting;
	}

protected:
	
	// 实际内部初始化 虚函数
	UE_API virtual void OnInitialize(ULocalPlayer* InLocalPlayer) PURE_VIRTUAL(, )

	// 当设置应用时
	virtual void OnSettingApplied(UGameSetting* Setting) { }

	// 注册设置
	UE_API void RegisterSetting(UGameSetting* InSetting);
	// 注册设置 内部执行
	UE_API void RegisterInnerSettings(UGameSetting* InSetting);

	// Internal event handlers.
	// 内部事件处理程序。

	// 处理设置改变
	UE_API void HandleSettingChanged(UGameSetting* Setting, EGameSettingChangeReason Reason);
	// 处理设置应用
	UE_API void HandleSettingApplied(UGameSetting* Setting);
	// 处理设置编辑条件发生改变
	UE_API void HandleSettingEditConditionsChanged(UGameSetting* Setting);
	// 处理动作事件
	UE_API void HandleSettingNamedAction(UGameSetting* Setting, FGameplayTag GameSettings_Action_Tag);
	// 处理导航事件
	UE_API void HandleSettingNavigation(UGameSetting* Setting);

	// 顶层的设置
	UPROPERTY(Transient)
	TArray<TObjectPtr<UGameSetting>> TopLevelSettings;

	// 已经注册的设置
	UPROPERTY(Transient)
	TArray<TObjectPtr<UGameSetting>> RegisteredSettings;

	// 指向的本地玩家类
	UPROPERTY(Transient)
	TObjectPtr<ULocalPlayer> OwningLocalPlayer;
};

```
## 获取游戏设置注册器

### 关于Outer是否会影响NewObject的生命周期
这个Get方法目前并未在游戏中使用.
注意着NewObject传递的ParentObject参数是不会影响Registry的生命周期的.
就是说OuterObject是不会影响生命周期的.这里是写了C++demo进行测试的.
所以一定要找个地方持有Registry.
``` cpp

	// 这个写法是错误的 因为会有GC问题 Outer并不能防止GC 当然也有可能它就是这样动态设计的
	// 实际上由控件蓝图Screen来实例化注册器并持有它
	static ULyraGameSettingRegistry* Get(ULyraLocalPlayer* InLocalPlayer);


```
``` cpp
	// 先寻找一下 是否有创建过了
	ULyraGameSettingRegistry* Registry = FindObject<ULyraGameSettingRegistry>(InLocalPlayer, TEXT("LyraGameSettingRegistry"), true);

	// 这个对象会由UMG持有 以下代码会造成误解!
	if (Registry == nullptr)
	{
		Registry = NewObject<ULyraGameSettingRegistry>(InLocalPlayer, TEXT("LyraGameSettingRegistry"));

		// 调用初始化
		Registry->Initialize(InLocalPlayer);
	}

	return Registry;

```
如果有兴趣的可以看一下FindObject的NewObecjt的底层代码

### NewObject
具体细节可以看大钊老师的深入理解UE5.关于UObject如何分配内存使用等过程.
``` cpp
FUNCTION_NON_NULL_RETURN_START
	T* NewObject(UObject* Outer, FName Name, EObjectFlags Flags = RF_NoFlags, UObject* Template = nullptr, bool bCopyTransientsFromClassDefaults = false, FObjectInstancingGraph* InInstanceGraph = nullptr, UPackage* InExternalPackage = nullptr)
FUNCTION_NON_NULL_RETURN_END
{
	// ...
	T* Result = static_cast<T*>(StaticConstructObject_Internal(Params));
	return Result;
}
```
``` cpp
UObject* StaticConstructObject_Internal(const FStaticConstructObjectParameters& Params)
{
	// ...
	Result = StaticAllocateObject(InClass, InOuter, InName, InFlags, Params.InternalSetFlags, bCanRecycleSubobjects, &bRecycledSubobject, Params.ExternalPackage, SerialNumber, RemoteId, &GCGuard);

	return Result;
}
```


``` cpp
// Global UObject array instance
FUObjectArray GUObjectArray;

```



``` cpp
UObject* StaticAllocateObject
(
	const UClass*	InClass,
	UObject*		InOuter,
	FName			InName,
	EObjectFlags	InFlags,
	EInternalObjectFlags InternalSetFlags,
	bool bCanRecycleSubobjects,
	bool* bOutRecycledSubobject,
	UPackage* ExternalPackage,
	int32 SerialNumber,
	FRemoteObjectId RemoteId,
	FGCReconstructionGuard* GCGuard)
{
		UE_AUTORTFM_OPEN_NO_VALIDATION
		{
			FMemory::Memzero((void *)Obj, TotalSize);
			new ((void *)Obj) UObjectBase(const_cast<UClass*>(InClass), InFlags|RF_NeedInitialization, InternalSetFlags, InOuter, InName, OldIndex, OldSerialNumber == 0 ? SerialNumber : OldSerialNumber, 
#if UE_WITH_REMOTE_OBJECT_HANDLE
				OldRemoteId.IsValid() ? OldRemoteId :
#endif
				RemoteId);
		};
}

```
``` cpp
UObjectBase::UObjectBase(UClass* InClass,
	EObjectFlags InFlags,
	EInternalObjectFlags InInternalFlags,
	UObject *InOuter,
	FName InName,
	int32 InInternalIndex,
	int32 InSerialNumber,
	FRemoteObjectId InRemoteId)
:	ObjectFlags			(InFlags)
,	InternalIndex		(INDEX_NONE)
,	ClassPrivate		(InClass)
,	OuterPrivate		(InOuter)
{
	check(ClassPrivate);
	// Add to global table.
	AddObject(InName, InInternalFlags, InInternalIndex, InSerialNumber, InRemoteId);
	
#if CSV_PROFILER_STATS && CSV_TRACK_UOBJECT_COUNT
	UObjectStats::IncrementUObjectCount();
#endif
}	

```


``` cpp
/**
 * Add a newly created object to the name hash tables and the object array
 *
 * @param Name name to assign to this uobject
 */
UE_AUTORTFM_ALWAYS_OPEN
void UObjectBase::AddObject(FName InName, EInternalObjectFlags InSetInternalFlags, int32 InInternalIndex, int32 InSerialNumber, FRemoteObjectId InRemoteId)
{
	NamePrivate = InName;
	EInternalObjectFlags InternalFlagsToSet = InSetInternalFlags;
	if (!IsInGameThread())
	{
		InternalFlagsToSet |= EInternalObjectFlags::Async;
	}
	if (ObjectFlags & RF_MarkAsRootSet)
	{		
		InternalFlagsToSet |= EInternalObjectFlags::RootSet;
		ObjectFlags &= ~RF_MarkAsRootSet;
	}
	if (ObjectFlags & RF_MarkAsNative)
	{
		InternalFlagsToSet |= EInternalObjectFlags::Native;
		ObjectFlags &= ~RF_MarkAsNative;
	}
	GUObjectArray.AllocateUObjectIndex(this, InternalFlagsToSet, InInternalIndex, InSerialNumber, InRemoteId);
	check(InName != NAME_None && InternalIndex >= 0);
	HashObject(this);
	check(IsValidLowLevel());
}

```

``` cpp
void FUObjectArray::AllocateUObjectIndex(UObjectBase* Object, EInternalObjectFlags InitialFlags, int32 AlreadyAllocatedIndex, int32 SerialNumber, FRemoteObjectId RemoteId)
{
	// ...

	Index = AlreadyAllocatedIndex;

	// ...
	// Add to global table.
	FUObjectItem* ObjectItem = IndexToObject(Index);
	ObjectItem->SetObject(Object);

}
```
### FindObeject

``` cpp
/**
 * Find an optional object.
 * @see StaticFindObject()
 */
template< class T > 
inline T* FindObject( UObject* Outer, const TCHAR* Name, bool ExactClass=false )
{
	return (T*)StaticFindObject( T::StaticClass(), Outer, Name, ExactClass );
}

```

``` cpp
//
// Find an optional object.
//
UObject* StaticFindObject( UClass* ObjectClass, UObject* InObjectPackage, const TCHAR* OrigInName, bool bExactClass )
{

	PRAGMA_DISABLE_DEPRECATION_WARNINGS
	return StaticFindObjectFast(ObjectClass, ObjectPackage, ObjectName, bExactClass, bAnyPackage);
	PRAGMA_ENABLE_DEPRECATION_WARNINGS
}
```
``` cpp

UObject* StaticFindObjectFast(UClass* ObjectClass, UObject* ObjectPackage, FName ObjectName, bool bExactClass, bool bAnyPackage, EObjectFlags ExclusiveFlags, EInternalObjectFlags ExclusiveInternalFlags)
{

	UObject* FoundObject = StaticFindObjectFastInternal(ObjectClass, ObjectPackage, ObjectName, 
	return FoundObject;
}
```
``` cpp
UObject* StaticFindObjectFastInternal(const UClass* ObjectClass, const UObject* ObjectPackage, FName ObjectName, bool bExactClass, bool bAnyPackage, EObjectFlags ExcludeFlags, EInternalObjectFlags ExclusiveInternalFlags)
{
	UObject* Result = nullptr;

	// Transactionally, a static find operation has to occur in the open as it touches shared state.
	UE_AUTORTFM_OPEN
	{
		INC_DWORD_STAT(STAT_FindObjectFast);

		check(!IS_ANY_PACKAGE_DEPRECATED(ObjectPackage)); // this could never have returned anything but nullptr

		// If they specified an outer use that during the hashing
		FUObjectHashTables& ThreadHash = FUObjectHashTables::Get();
		Result = StaticFindObjectFastInternalThreadSafe(ThreadHash, ObjectClass, ObjectPackage, ObjectName, bExactClass, bAnyPackage, ExcludeFlags, ExclusiveInternalFlags);
	};

	return Result;
}

```
``` cpp
UObject* StaticFindObjectFastInternalThreadSafe(FUObjectHashTables& ThreadHash, const UClass* ObjectClass, const UObject* ObjectPackage, FName ObjectName, bool bExactClass, bool bAnyPackage, EObjectFlags ExcludeFlags, EInternalObjectFlags ExclusiveInternalFlags)
{
	ExclusiveInternalFlags |= DefaultInternalExclusionFlags;

	// If they specified an outer use that during the hashing
	UObject* Result = nullptr;
	if (ObjectPackage != nullptr)
	{
		int32 Hash = GetObjectOuterHash(ObjectName, (PTRINT)ObjectPackage);
		FHashTableLock HashLock(ThreadHash);
		for (TMultiMap<int32, uint32>::TConstKeyIterator HashIt(ThreadHash.HashOuter, Hash); HashIt; ++HashIt)
		{
			uint32 InternalIndex = HashIt.Value();
			UObject* Object = static_cast<UObject*>(GUObjectArray.IndexToObject(InternalIndex)->GetObject());
			if
				/* check that the name matches the name we're searching for */
				((Object->GetFName() == ObjectName)

				/* Don't return objects that have any of the exclusive flags set */
				&& !Object->HasAnyFlags(ExcludeFlags)

				/* check that the object has the correct Outer */
				&& Object->GetOuter() == ObjectPackage

				/** If a class was specified, check that the object is of the correct class */
				&& (ObjectClass == nullptr || (bExactClass ? Object->GetClass() == ObjectClass : Object->IsA(ObjectClass)))
				
				/** Include (or not) pending kill objects */
				&& !Object->HasAnyInternalFlags(ExclusiveInternalFlags))
			{
				checkf(!Object->IsUnreachable(), TEXT("%s"), *Object->GetFullName());
				if (Result)
				{
					UE_LOG(LogUObjectHash, Warning, TEXT("Ambiguous search, could be %s or %s"), *GetFullNameSafe(Result), *GetFullNameSafe(Object));
				}
				else
				{
					Result = Object;
				}
#if (UE_BUILD_SHIPPING || UE_BUILD_TEST)
				break;
#endif
			}
		}

#if WITH_EDITOR
		// if the search fail and the OuterPackage is a UPackage, lookup potential external package
		if (Result == nullptr && ObjectPackage->IsA(UPackage::StaticClass()))
		{
			Result = StaticFindObjectInPackageInternal(ThreadHash, ObjectClass, static_cast<const UPackage*>(ObjectPackage), ObjectName, bExactClass, ExcludeFlags, ExclusiveInternalFlags);
		}
#endif
	}
	else
	{
		FObjectSearchPath SearchPath(ObjectName);

		const int32 Hash = GetObjectHash(SearchPath.Inner);
		FHashTableLock HashLock(ThreadHash);

		FHashBucket* Bucket = ThreadHash.Hash.Find(Hash);
		if (Bucket)
		{
			for (FHashBucketIterator It(*Bucket); It; ++It)
			{
				UObject* Object = (UObject*)*It;

				if
					((Object->GetFName() == SearchPath.Inner)

					/* Don't return objects that have any of the exclusive flags set */
					&& !Object->HasAnyFlags(ExcludeFlags)

					/*If there is no package (no InObjectPackage specified, and InName's package is "")
					and the caller specified any_package, then accept it, regardless of its package.
					Or, if the object is a top-level package then accept it immediately.*/
					&& (bAnyPackage || !Object->GetOuter())

					/** If a class was specified, check that the object is of the correct class */
					&& (ObjectClass == nullptr || (bExactClass ? Object->GetClass() == ObjectClass : Object->IsA(ObjectClass)))

					/** Include (or not) pending kill objects */
					&& !Object->HasAnyInternalFlags(ExclusiveInternalFlags)

					/** Ensure that the partial path provided matches the object found */
					&& SearchPath.MatchOuterNames(Object->GetOuter()))
				{
					checkf(!Object->IsUnreachable(), TEXT("%s"), *Object->GetFullName());
					if (Result)
					{
						UE_LOG(LogUObjectHash, Warning, TEXT("Ambiguous path search, could be %s or %s"), *GetFullNameSafe(Result), *GetFullNameSafe(Object));
					}
					else
					{
						Result = Object;
					}
#if (UE_BUILD_SHIPPING || UE_BUILD_TEST)
					break;
#endif
				}
			}
		}
	}
	// Not found.
	if (Result && UE::GC::GIsIncrementalReachabilityPending)
	{
		UE::GC::MarkAsReachable(Result);
	}
	return Result;
}
```

### 测试Outer的生命周期代码
蓝图在调用测试方法后 再调用蓝图强制GC的节点即可!
``` cpp
UCLASS()
class UXGGCParentObject:public UObject
{
	GENERATED_BODY()
	
public:
	
	UXGGCParentObject()
	{
		UE_LOG(LogTemp,Error,TEXT("父Object:我被构造了"));
	}

	virtual ~UXGGCParentObject()
	{
		UE_LOG(LogTemp,Error,TEXT("父Object:我析构了"));
	}
};

UCLASS()
class UXGGCObject:public UObject
{
	GENERATED_BODY()
	
public:
	
	UXGGCObject()
	{
		UE_LOG(LogTemp,Error,TEXT("子Object:我被构造了"));
	}

	virtual ~UXGGCObject()
	{
		UE_LOG(LogTemp,Error,TEXT("子Object:我析构了"));
	}
};

UCLASS()
class AMYActor:public AActor
{
	GENERATED_BODY()
public:

	virtual	void BeginPlay() override
	{
		Super::BeginPlay();

		UE_LOG(LogTemp,Error,TEXT("AMyActor:我被创建了"));

		UXGGCObject* MyObject = NewObject<UXGGCObject>(this,UXGGCObject::StaticClass(),TEXT("XGGCObject"));
	
	}
};



/**
 * 
 */
UCLASS()
class GCDEMO_API UMyClass : public UBlueprintFunctionLibrary
{
	GENERATED_BODY()

public:

	UFUNCTION(BlueprintCallable,Category="XGTest")
 static	void TestMethod();

	
	UFUNCTION(BlueprintCallable,Category="XGTest")
	static void TestMethod2();
};

```
``` cpp

void UMyClass::TestMethod()
{
	UXGGCParentObject* RootObejct = NewObject<UXGGCParentObject>();
	RootObejct->AddToRoot();
	
	UXGGCObject* MyObject = NewObject<UXGGCObject>(RootObejct,UXGGCObject::StaticClass(),TEXT("XGGCObject"));
	
	
}

void UMyClass::TestMethod2()
{
	UXGGCParentObject* RootObejct = NewObject<UXGGCParentObject>();

	UXGGCObject* MyObject = NewObject<UXGGCObject>(RootObejct,UXGGCObject::StaticClass(),TEXT("XGGCObject"));
	
}

```
## 真正获取初始化游戏设置注册器
再次强调一定要持有!而且有效的持有!
``` cpp
UGameSettingRegistry* ULyraSettingScreen::CreateRegistry()
{
	ULyraGameSettingRegistry* NewRegistry = NewObject<ULyraGameSettingRegistry>();

	if (ULyraLocalPlayer* LocalPlayer = CastChecked<ULyraLocalPlayer>(GetOwningLocalPlayer()))
	{
		// 极其重要
		NewRegistry->Initialize(LocalPlayer);
	}

	return NewRegistry;
}

```

## 初始化游戏设置

### 初始化流程
``` cpp
void UGameSettingRegistry::Initialize(ULocalPlayer* InLocalPlayer)
{
	OwningLocalPlayer = InLocalPlayer;
	OnInitialize(InLocalPlayer);

	//UGameFeaturesSubsystem
}

```

``` cpp
void ULyraGameSettingRegistry::OnInitialize(ULocalPlayer* InLocalPlayer)
{
	ULyraLocalPlayer* LyraLocalPlayer = Cast<ULyraLocalPlayer>(InLocalPlayer);

	// 初始化视频设置
	VideoSettings = InitializeVideoSettings(LyraLocalPlayer);
	
	// 填充帧率限制选项
	InitializeVideoSettings_FrameRates(VideoSettings, LyraLocalPlayer);
	// 这里是注册到了顶层中 并递归注册到所有注册设置中.只有它的顶层才注册到顶层中!
	RegisterSetting(VideoSettings);


	//初始化音频设置
	AudioSettings = InitializeAudioSettings(LyraLocalPlayer);
	RegisterSetting(AudioSettings);

	// 初始化游戏设置
	GameplaySettings = InitializeGameplaySettings(LyraLocalPlayer); 
	RegisterSetting(GameplaySettings);

	// 键鼠设置
	MouseAndKeyboardSettings = InitializeMouseAndKeyboardSettings(LyraLocalPlayer);
	RegisterSetting(MouseAndKeyboardSettings);

	// 手柄设置
	GamepadSettings = InitializeGamepadSettings(LyraLocalPlayer);
	RegisterSetting(GamepadSettings);
}

```
### 创建游戏设置集合
此处以最简单的音频设置为例
``` cpp

UGameSettingCollection* ULyraGameSettingRegistry::InitializeAudioSettings(ULyraLocalPlayer* InLocalPlayer)
{
	// 创建音频界面屏幕
	UGameSettingCollection* Screen = NewObject<UGameSettingCollection>();
	Screen->SetDevName(TEXT("AudioCollection"));
	Screen->SetDisplayName(LOCTEXT("AudioCollection_Name", "Audio"));
	Screen->Initialize(InLocalPlayer);

	// Volume
	// 音量
	////////////////////////////////////////////////////////////////////////////////////
	{
		// 创建音量的设置集合
		UGameSettingCollection* Volume = NewObject<UGameSettingCollection>();
		Volume->SetDevName(TEXT("VolumeCollection"));
		Volume->SetDisplayName(LOCTEXT("VolumeCollection_Name", "Volume"));
		Screen->AddSetting(Volume);

		//----------------------------------------------------------------------------------
		{
			// 整体音量
			UGameSettingValueScalarDynamic* Setting = NewObject<UGameSettingValueScalarDynamic>();
			Setting->SetDevName(TEXT("OverallVolume"));
			Setting->SetDisplayName(LOCTEXT("OverallVolume_Name", "Overall"));
			Setting->SetDescriptionRichText(LOCTEXT("OverallVolume_Description", "Adjusts the volume of everything."));

			Setting->SetDynamicGetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(GetOverallVolume));
			Setting->SetDynamicSetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(SetOverallVolume));
			Setting->SetDefaultValue(GetDefault<ULyraSettingsLocal>()->GetOverallVolume());
			Setting->SetDisplayFormat(UGameSettingValueScalarDynamic::ZeroToOnePercent);

			Setting->AddEditCondition(FWhenPlayingAsPrimaryPlayer::Get());

			Volume->AddSetting(Setting);
		}
		//----------------------------------------------------------------------------------
		{
			// 音乐音量
			
			UGameSettingValueScalarDynamic* Setting = NewObject<UGameSettingValueScalarDynamic>();
			Setting->SetDevName(TEXT("MusicVolume"));
			Setting->SetDisplayName(LOCTEXT("MusicVolume_Name", "Music"));
			Setting->SetDescriptionRichText(LOCTEXT("MusicVolume_Description", "Adjusts the volume of music."));

			Setting->SetDynamicGetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(GetMusicVolume));
			Setting->SetDynamicSetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(SetMusicVolume));
			Setting->SetDefaultValue(GetDefault<ULyraSettingsLocal>()->GetMusicVolume());
			Setting->SetDisplayFormat(UGameSettingValueScalarDynamic::ZeroToOnePercent);

			Setting->AddEditCondition(FWhenPlayingAsPrimaryPlayer::Get());

			Volume->AddSetting(Setting);
		}
		//----------------------------------------------------------------------------------
		{
			// 音效音量
			
			UGameSettingValueScalarDynamic* Setting = NewObject<UGameSettingValueScalarDynamic>();
			Setting->SetDevName(TEXT("SoundEffectsVolume"));
			Setting->SetDisplayName(LOCTEXT("SoundEffectsVolume_Name", "Sound Effects"));
			Setting->SetDescriptionRichText(LOCTEXT("SoundEffectsVolume_Description", "Adjusts the volume of sound effects."));

			Setting->SetDynamicGetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(GetSoundFXVolume));
			Setting->SetDynamicSetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(SetSoundFXVolume));
			Setting->SetDefaultValue(GetDefault<ULyraSettingsLocal>()->GetSoundFXVolume());
			Setting->SetDisplayFormat(UGameSettingValueScalarDynamic::ZeroToOnePercent);

			Setting->AddEditCondition(FWhenPlayingAsPrimaryPlayer::Get());

			Volume->AddSetting(Setting);
		}
		//----------------------------------------------------------------------------------
		{
			// 对话音量
			
			UGameSettingValueScalarDynamic* Setting = NewObject<UGameSettingValueScalarDynamic>();
			Setting->SetDevName(TEXT("DialogueVolume"));
			Setting->SetDisplayName(LOCTEXT("DialogueVolume_Name", "Dialogue"));
			Setting->SetDescriptionRichText(LOCTEXT("DialogueVolume_Description", "Adjusts the volume of dialogue for game characters and voice overs."));

			Setting->SetDynamicGetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(GetDialogueVolume));
			Setting->SetDynamicSetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(SetDialogueVolume));
			Setting->SetDefaultValue(GetDefault<ULyraSettingsLocal>()->GetDialogueVolume());
			Setting->SetDisplayFormat(UGameSettingValueScalarDynamic::ZeroToOnePercent);

			Setting->AddEditCondition(FWhenPlayingAsPrimaryPlayer::Get());

			Volume->AddSetting(Setting);
		}
		//----------------------------------------------------------------------------------
		{
			// 聊天音量
			
			UGameSettingValueScalarDynamic* Setting = NewObject<UGameSettingValueScalarDynamic>();
			Setting->SetDevName(TEXT("VoiceChatVolume"));
			Setting->SetDisplayName(LOCTEXT("VoiceChatVolume_Name", "Voice Chat"));
			Setting->SetDescriptionRichText(LOCTEXT("VoiceChatVolume_Description", "Adjusts the volume of voice chat."));

			Setting->SetDynamicGetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(GetVoiceChatVolume));
			Setting->SetDynamicSetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(SetVoiceChatVolume));
			Setting->SetDefaultValue(GetDefault<ULyraSettingsLocal>()->GetVoiceChatVolume());
			Setting->SetDisplayFormat(UGameSettingValueScalarDynamic::ZeroToOnePercent);

			Setting->AddEditCondition(FWhenPlayingAsPrimaryPlayer::Get());

			Volume->AddSetting(Setting);
		}
	}


	// Sound
	// 音效
	////////////////////////////////////////////////////////////////////////////////////
	{
		UGameSettingCollection* Sound = NewObject<UGameSettingCollection>();
		Sound->SetDevName(TEXT("SoundCollection"));
		Sound->SetDisplayName(LOCTEXT("SoundCollection_Name", "Sound"));
		Screen->AddSetting(Sound);

		//----------------------------------------------------------------------------------
		{
			// 字幕页
			UGameSettingCollectionPage* SubtitlePage = NewObject<UGameSettingCollectionPage>();
			SubtitlePage->SetDevName(TEXT("SubtitlePage"));
			SubtitlePage->SetDisplayName(LOCTEXT("SubtitlePage_Name", "Subtitles"));
			SubtitlePage->SetDescriptionRichText(LOCTEXT("SubtitlePage_Description", "Configure the visual appearance of subtitles."));
			SubtitlePage->SetNavigationText(LOCTEXT("SubtitlePage_Navigation", "Options"));

			SubtitlePage->AddEditCondition(FWhenPlayingAsPrimaryPlayer::Get());

			Sound->AddSetting(SubtitlePage);

			// Subtitles
			// 字幕
			////////////////////////////////////////////////////////////////////////////////////
			{
				UGameSettingCollection* SubtitleCollection = NewObject<UGameSettingCollection>();
				SubtitleCollection->SetDevName(TEXT("SubtitlesCollection"));
				SubtitleCollection->SetDisplayName(LOCTEXT("SubtitlesCollection_Name", "Subtitles"));
				SubtitlePage->AddSetting(SubtitleCollection);

				//----------------------------------------------------------------------------------
				{
					// 字幕开启关闭
					UGameSettingValueDiscreteDynamic_Bool* Setting = NewObject<UGameSettingValueDiscreteDynamic_Bool>();
					Setting->SetDevName(TEXT("Subtitles"));
					Setting->SetDisplayName(LOCTEXT("Subtitles_Name", "Subtitles"));
					Setting->SetDescriptionRichText(LOCTEXT("Subtitles_Description", "Turns subtitles on/off."));

					Setting->SetDynamicGetter(GET_SHARED_SETTINGS_FUNCTION_PATH(GetSubtitlesEnabled));
					Setting->SetDynamicSetter(GET_SHARED_SETTINGS_FUNCTION_PATH(SetSubtitlesEnabled));
					Setting->SetDefaultValue(GetDefault<ULyraSettingsShared>()->GetSubtitlesEnabled());

					SubtitleCollection->AddSetting(Setting);
				}
				//----------------------------------------------------------------------------------
				{
					// 字体大小
					
					UGameSettingValueDiscreteDynamic_Enum* Setting = NewObject<UGameSettingValueDiscreteDynamic_Enum>();
					Setting->SetDevName(TEXT("SubtitleTextSize"));
					Setting->SetDisplayName(LOCTEXT("SubtitleTextSize_Name", "Text Size"));
					Setting->SetDescriptionRichText(LOCTEXT("SubtitleTextSize_Description", "Choose different sizes of the the subtitle text."));

					Setting->SetDynamicGetter(GET_SHARED_SETTINGS_FUNCTION_PATH(GetSubtitlesTextSize));
					Setting->SetDynamicSetter(GET_SHARED_SETTINGS_FUNCTION_PATH(SetSubtitlesTextSize));
					Setting->SetDefaultValue(GetDefault<ULyraSettingsShared>()->GetSubtitlesTextSize());
					Setting->AddEnumOption(ESubtitleDisplayTextSize::ExtraSmall, LOCTEXT("ESubtitleTextSize_ExtraSmall", "Extra Small"));
					Setting->AddEnumOption(ESubtitleDisplayTextSize::Small, LOCTEXT("ESubtitleTextSize_Small", "Small"));
					Setting->AddEnumOption(ESubtitleDisplayTextSize::Medium, LOCTEXT("ESubtitleTextSize_Medium", "Medium"));
					Setting->AddEnumOption(ESubtitleDisplayTextSize::Large, LOCTEXT("ESubtitleTextSize_Large", "Large"));
					Setting->AddEnumOption(ESubtitleDisplayTextSize::ExtraLarge, LOCTEXT("ESubtitleTextSize_ExtraLarge", "Extra Large"));

					SubtitleCollection->AddSetting(Setting);
				}
				//----------------------------------------------------------------------------------
				{
					// 字体颜色
					
					UGameSettingValueDiscreteDynamic_Enum* Setting = NewObject<UGameSettingValueDiscreteDynamic_Enum>();
					Setting->SetDevName(TEXT("SubtitleTextColor"));
					Setting->SetDisplayName(LOCTEXT("SubtitleTextColor_Name", "Text Color"));
					Setting->SetDescriptionRichText(LOCTEXT("SubtitleTextColor_Description", "Choose different colors for the subtitle text."));

					Setting->SetDynamicGetter(GET_SHARED_SETTINGS_FUNCTION_PATH(GetSubtitlesTextColor));
					Setting->SetDynamicSetter(GET_SHARED_SETTINGS_FUNCTION_PATH(SetSubtitlesTextColor));
					Setting->SetDefaultValue(GetDefault<ULyraSettingsShared>()->GetSubtitlesTextColor());
					Setting->AddEnumOption(ESubtitleDisplayTextColor::White, LOCTEXT("ESubtitleTextColor_White", "White"));
					Setting->AddEnumOption(ESubtitleDisplayTextColor::Yellow, LOCTEXT("ESubtitleTextColor_Yellow", "Yellow"));

					SubtitleCollection->AddSetting(Setting);
				}
				//----------------------------------------------------------------------------------
				{
					// 字体边框
					
					UGameSettingValueDiscreteDynamic_Enum* Setting = NewObject<UGameSettingValueDiscreteDynamic_Enum>();
					Setting->SetDevName(TEXT("SubtitleTextBorder"));
					Setting->SetDisplayName(LOCTEXT("SubtitleBackgroundStyle_Name", "Text Border"));
					Setting->SetDescriptionRichText(LOCTEXT("SubtitleTextBorder_Description", "Choose different borders for the text."));

					Setting->SetDynamicGetter(GET_SHARED_SETTINGS_FUNCTION_PATH(GetSubtitlesTextBorder));
					Setting->SetDynamicSetter(GET_SHARED_SETTINGS_FUNCTION_PATH(SetSubtitlesTextBorder));
					Setting->SetDefaultValue(GetDefault<ULyraSettingsShared>()->GetSubtitlesTextBorder());
					Setting->AddEnumOption(ESubtitleDisplayTextBorder::None, LOCTEXT("ESubtitleTextBorder_None", "None"));
					Setting->AddEnumOption(ESubtitleDisplayTextBorder::Outline, LOCTEXT("ESubtitleTextBorder_Outline", "Outline"));
					Setting->AddEnumOption(ESubtitleDisplayTextBorder::DropShadow, LOCTEXT("ESubtitleTextBorder_DropShadow", "Drop Shadow"));

					SubtitleCollection->AddSetting(Setting);
				}
				//----------------------------------------------------------------------------------
				{
					// 背景透明度
					
					UGameSettingValueDiscreteDynamic_Enum* Setting = NewObject<UGameSettingValueDiscreteDynamic_Enum>();
					Setting->SetDevName(TEXT("SubtitleBackgroundOpacity"));
					Setting->SetDisplayName(LOCTEXT("SubtitleBackground_Name", "Background Opacity"));
					Setting->SetDescriptionRichText(LOCTEXT("SubtitleBackgroundOpacity_Description", "Choose a different background or letterboxing for the subtitles."));

					Setting->SetDynamicGetter(GET_SHARED_SETTINGS_FUNCTION_PATH(GetSubtitlesBackgroundOpacity));
					Setting->SetDynamicSetter(GET_SHARED_SETTINGS_FUNCTION_PATH(SetSubtitlesBackgroundOpacity));
					Setting->SetDefaultValue(GetDefault<ULyraSettingsShared>()->GetSubtitlesBackgroundOpacity());
					Setting->AddEnumOption(ESubtitleDisplayBackgroundOpacity::Clear, LOCTEXT("ESubtitleBackgroundOpacity_Clear", "Clear"));
					Setting->AddEnumOption(ESubtitleDisplayBackgroundOpacity::Low, LOCTEXT("ESubtitleBackgroundOpacity_Low", "Low"));
					Setting->AddEnumOption(ESubtitleDisplayBackgroundOpacity::Medium, LOCTEXT("ESubtitleBackgroundOpacity_Medium", "Medium"));
					Setting->AddEnumOption(ESubtitleDisplayBackgroundOpacity::High, LOCTEXT("ESubtitleBackgroundOpacity_High", "High"));
					Setting->AddEnumOption(ESubtitleDisplayBackgroundOpacity::Solid, LOCTEXT("ESubtitleBackgroundOpacity_Solid", "Solid"));

					SubtitleCollection->AddSetting(Setting);
				}
			}
		}
		//----------------------------------------------------------------------------------
		{
			// 音频设备切换
			
			ULyraSettingValueDiscreteDynamic_AudioOutputDevice* Setting = NewObject<ULyraSettingValueDiscreteDynamic_AudioOutputDevice>();
			Setting->SetDevName(TEXT("AudioOutputDevice"));
			Setting->SetDisplayName(LOCTEXT("AudioOutputDevice_Name", "Audio Output Device"));
			Setting->SetDescriptionRichText(LOCTEXT("AudioOutputDevice_Description", "Changes the audio output device for game audio (not voice chat)."));

			Setting->SetDynamicGetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(GetAudioOutputDeviceId));
			Setting->SetDynamicSetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(SetAudioOutputDeviceId));

			Setting->AddEditCondition(FWhenPlayingAsPrimaryPlayer::Get());
			Setting->AddEditCondition(FWhenPlatformHasTrait::KillIfMissing(
				TAG_Platform_Trait_SupportsChangingAudioOutputDevice,
				TEXT("Platform does not support changing audio output device"))
			);

			Sound->AddSetting(Setting);
		}
		//----------------------------------------------------------------------------------
		{
			// 后台音乐开启
			
			UGameSettingValueDiscreteDynamic_Enum* Setting = NewObject<UGameSettingValueDiscreteDynamic_Enum>();
			Setting->SetDevName(TEXT("BackgroundAudio"));
			Setting->SetDisplayName(LOCTEXT("BackgroundAudio_Name", "Background Audio"));
			Setting->SetDescriptionRichText(LOCTEXT("BackgroundAudio_Description", "Turns game audio on/off when the game is in the background. When on, the game audio will continue to play when the game is minimized, or another window is focused."));

			Setting->SetDynamicGetter(GET_SHARED_SETTINGS_FUNCTION_PATH(GetAllowAudioInBackgroundSetting));
			Setting->SetDynamicSetter(GET_SHARED_SETTINGS_FUNCTION_PATH(SetAllowAudioInBackgroundSetting));
			Setting->SetDefaultValue(GetDefault<ULyraSettingsShared>()->GetAllowAudioInBackgroundSetting());

			Setting->AddEnumOption(ELyraAllowBackgroundAudioSetting::Off, LOCTEXT("ELyraAllowBackgroundAudioSetting_Off", "Off"));
			Setting->AddEnumOption(ELyraAllowBackgroundAudioSetting::AllSounds, LOCTEXT("ELyraAllowBackgroundAudioSetting_AllSounds", "All Sounds"));

			Setting->AddEditCondition(FWhenPlayingAsPrimaryPlayer::Get());
			Setting->AddEditCondition(FWhenPlatformHasTrait::KillIfMissing(
				TAG_Platform_Trait_SupportsBackgroundAudio,
				TEXT("Platform does not support background audio"))
			);

			Sound->AddSetting(Setting);
		}
		//----------------------------------------------------------------------------------
		{
			// 耳机模式
			
			UGameSettingValueDiscreteDynamic_Bool* Setting = NewObject<UGameSettingValueDiscreteDynamic_Bool>();
			Setting->SetDevName(TEXT("HeadphoneMode"));
			Setting->SetDisplayName(LOCTEXT("HeadphoneMode_Name", "3D Headphones"));
			Setting->SetDescriptionRichText(LOCTEXT("HeadphoneMode_Description", "Enable binaural audio.  Provides 3D audio spatialization, so you can hear the location of sounds more precisely, including above, below, and behind you. Recommended for use with stereo headphones only."));

			Setting->SetDynamicGetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(bDesiredHeadphoneMode));
			Setting->SetDynamicSetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(bDesiredHeadphoneMode));
			Setting->SetDefaultValue(GetDefault<ULyraSettingsLocal>()->IsHeadphoneModeEnabled());

			Setting->AddEditCondition(MakeShared<FWhenCondition>(
				[](const ULocalPlayer*, FGameSettingEditableState& InOutEditState)
				{
					if (!GetDefault<ULyraSettingsLocal>()->CanModifyHeadphoneModeEnabled())
					{
						InOutEditState.Kill(TEXT("Binaural Spatialization option cannot be modified on this platform"));
					}
				}));

			Setting->AddEditCondition(FWhenPlayingAsPrimaryPlayer::Get());

			Sound->AddSetting(Setting);
		}
		//----------------------------------------------------------------------------------
		{
			// HDR音频
			
			UGameSettingValueDiscreteDynamic_Bool* Setting = NewObject<UGameSettingValueDiscreteDynamic_Bool>();
			Setting->SetDevName(TEXT("HDRAudioMode"));
			Setting->SetDisplayName(LOCTEXT("HDRAudioMode_Name", "High Dynamic Range Audio"));
			Setting->SetDescriptionRichText(LOCTEXT("HDRAudioMode_Description", "Enable high dynamic range audio. Changes the runtime processing chain to increase the dynamic range of the audio mixdown, appropriate for theater or more cinematic experiences."));

			Setting->SetDynamicGetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(bUseHDRAudioMode));
			Setting->SetDynamicSetter(GET_LOCAL_SETTINGS_FUNCTION_PATH(SetHDRAudioModeEnabled));
			Setting->SetDefaultValue(GetDefault<ULyraSettingsLocal>()->IsHDRAudioModeEnabled());

			Setting->AddEditCondition(FWhenPlayingAsPrimaryPlayer::Get());

			Sound->AddSetting(Setting);
		}
		//----------------------------------------------------------------------------------
	}

	return Screen;
}

```


### Get/Set方法
宏定义,主要是去找我们的本地游戏设置类
``` cpp
// Copyright Epic Games, Inc. All Rights Reserved.
// Finished
// 001 直接完成即可.
#pragma once


#include "DataSource/GameSettingDataSourceDynamic.h" // IWYU pragma: keep
#include "GameSettingRegistry.h"
#include "Settings/LyraSettingsLocal.h" // IWYU pragma: keep

#include "LyraGameSettingRegistry.generated.h"

class ULocalPlayer;
class UObject;

// 校验get方法 返回函数或者属性的名称
#define GET_SHARED_SETTINGS_FUNCTION_PATH(FunctionOrPropertyName)							\
	MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({								\
		GET_FUNCTION_NAME_STRING_CHECKED(ULyraLocalPlayer, GetSharedSettings),				\
		GET_FUNCTION_NAME_STRING_CHECKED(ULyraSettingsShared, FunctionOrPropertyName)		\
	}))
// 校验set方法 返回函数或者属性的名称
#define GET_LOCAL_SETTINGS_FUNCTION_PATH(FunctionOrPropertyName)							\
	MakeShared<FGameSettingDataSourceDynamic>(TArray<FString>({								\
		GET_FUNCTION_NAME_STRING_CHECKED(ULyraLocalPlayer, GetLocalSettings),				\
		GET_FUNCTION_NAME_STRING_CHECKED(ULyraSettingsLocal, FunctionOrPropertyName)		\
	}))

#define GET_FUNCTION_NAME_STRING_CHECKED(ClassName, FunctionName) \
	((void)sizeof(&ClassName::FunctionName), TEXT(#FunctionName))

```
此处见拓展文档UE5_Lyra学习指南_055_泽_01_IWYU
此处见拓展文档UE5_Lyra学习指南_055_泽_02_Get方法编译安全检测
## 数据源绑定
``` cpp
//--------------------------------------
// FGameSettingDataSource
//--------------------------------------

class FGameSettingDataSource : public TSharedFromThis<FGameSettingDataSource>
{
public:
	virtual ~FGameSettingDataSource() { }

	/**
	 * Some settings may take an async amount of time to finish initializing.  The settings system will wait
	 * for all settings to be ready before showing the setting.
	 *
	 *  有些设置的初始化过程可能会需要相当长的时间。设置系统会一直等待所有设置都准备就绪后，才会显示这些设置
	 */
	virtual void Startup(ULocalPlayer* InLocalPlayer, FSimpleDelegate StartupCompleteCallback) { StartupCompleteCallback.ExecuteIfBound(); }

	// 解算是否成功
	virtual bool Resolve(ULocalPlayer* InContext) = 0;

	
	virtual FString GetValueAsString(ULocalPlayer* InContext) const = 0;

	virtual void SetValue(ULocalPlayer* InContext, const FString& Value) = 0;

	virtual FString ToString() const = 0;
};

```
``` cpp
//--------------------------------------
// FGameSettingDataSourceDynamic
//--------------------------------------

class FGameSettingDataSourceDynamic : public FGameSettingDataSource
{
public:
	UE_API FGameSettingDataSourceDynamic(const TArray<FString>& InDynamicPath);

	UE_API virtual bool Resolve(ULocalPlayer* InLocalPlayer) override;

	UE_API virtual FString GetValueAsString(ULocalPlayer* InLocalPlayer) const override;

	UE_API virtual void SetValue(ULocalPlayer* InLocalPlayer, const FString& Value) override;

	UE_API virtual FString ToString() const override;

private:
	FCachedPropertyPath DynamicPath;
};

```
### 注册路径
``` cpp
FGameSettingDataSourceDynamic::FGameSettingDataSourceDynamic(const TArray<FString>& InDynamicPath)
	: DynamicPath(InDynamicPath)
{
}


```
``` cpp
FCachedPropertyPath::FCachedPropertyPath(const TArray<FString>& PathSegments)
	: CachedAddress(nullptr)
	, CachedFunction(nullptr)
	, CachedContainer(nullptr)
	, CachedLastContainerInPath(nullptr)
	, CachedLastContainerInPathIndex(INDEX_NONE)
	, bCanSafelyUsedCachedAddress(false)
{
	for (const FString& Segment : PathSegments)
	{
		Segments.Add(FPropertyPathSegment(Segment.Len(), *Segment));
	}
}

```
### 解算路径
``` cpp
bool FGameSettingDataSourceDynamic::Resolve(ULocalPlayer* InLocalPlayer)
{
	return DynamicPath.Resolve(InLocalPlayer);
}


```
``` cpp
bool FCachedPropertyPath::Resolve(UObject* InContainer) const
{
	FInternalCacheResolver Resolver;
	return PropertyPathHelpersInternal::ResolvePropertyPath(InContainer, *this, Resolver);
}

/** Helper for cache/copy resolver */
struct FInternalCacheResolver : public PropertyPathHelpersInternal::TPropertyPathResolver<FInternalCacheResolver>
{
	template<typename ContainerType>
	bool Resolve_Impl(ContainerType* InContainer, const FCachedPropertyPath& InPropertyPath)
	{
		return PropertyPathHelpersInternal::CacheResolveAddress(InContainer, InPropertyPath);
	}
};
```
``` cpp
	bool ResolvePropertyPath(UObject* InContainer, const FCachedPropertyPath& InPropertyPath, FPropertyPathResolver& InResolver)
	{
		if (InContainer)
		{
			return IteratePropertyPathRecursive<UObject>(InContainer->GetClass(), InContainer, 0, InPropertyPath, InResolver);
		}

		return false;
	}


```
注意下面这个函数比较长.并且递归了.
实际上就是把路径的数组一个一个拼上去进行访问.直到访问到最后一个.
注意路径可能是属性,也可能是方法!
``` cpp
template<typename ContainerType>
	static bool IteratePropertyPathRecursive(UStruct* InStruct, ContainerType* InContainer, int32 SegmentIndex, const FCachedPropertyPath& InPropertyPath, FPropertyPathResolver& InResolver)
	{
		const FPropertyPathSegment& Segment = InPropertyPath.GetSegment(SegmentIndex);
		const int32 ArrayIndex = Segment.GetArrayIndex() == INDEX_NONE ? 0 : Segment.GetArrayIndex();

		// Reset cached address usage flag at the path root. This will be reset later in the recursion if conditions are not met in the path.
		if(SegmentIndex == 0)
		{
			InPropertyPath.SetCachedContainer(InContainer);
			InPropertyPath.SetCanSafelyUsedCachedAddress(true);
		}

		// Obtain the property info from the given structure definition
		FFieldVariant Field = Segment.Resolve(InStruct);
		if (Field.IsValid())
		{
			const bool bFinalSegment = SegmentIndex == (InPropertyPath.GetNumSegments() - 1);

			if ( FProperty* Property = Field.Get<FProperty>() )
			{
				if (bFinalSegment)
				{
					return InResolver.Resolve(static_cast<ContainerType*>(InContainer), InPropertyPath);
				}
				else
				{
					// Check first to see if this is a simple object (eg. not an array of objects)
					if ( FObjectProperty* ObjectProperty = CastField<FObjectProperty>(Property) )
					{
						// Object boundary that can change, so we cant use the cached address safely
						InPropertyPath.SetCanSafelyUsedCachedAddress(false);

						// If it's an object we need to get the value of the property in the container first before we 
						// can continue, if the object is null we safely stop processing the chain of properties.
						if ( UObject* CurrentObject = ObjectProperty->GetPropertyValue_InContainer(InContainer, ArrayIndex) )
						{
							InPropertyPath.SetCachedLastContainer(CurrentObject, SegmentIndex);
							return IteratePropertyPathRecursive(CurrentObject->GetClass(), CurrentObject, SegmentIndex + 1, InPropertyPath, InResolver);
						}
					}
					// Check to see if this is a simple weak object property (eg. not an array of weak objects).
					if ( FWeakObjectProperty* WeakObjectProperty = CastField<FWeakObjectProperty>(Property) )
					{
						// Object boundary that can change, so we cant use the cached address safely
						InPropertyPath.SetCanSafelyUsedCachedAddress(false);

						FWeakObjectPtr WeakObject = WeakObjectProperty->GetPropertyValue_InContainer(InContainer, ArrayIndex);

						// If it's an object we need to get the value of the property in the container first before we 
						// can continue, if the object is null we safely stop processing the chain of properties.
						if ( UObject* CurrentObject = WeakObject.Get() )
						{
							InPropertyPath.SetCachedLastContainer(CurrentObject, SegmentIndex);
							return IteratePropertyPathRecursive(CurrentObject->GetClass(), CurrentObject, SegmentIndex + 1, InPropertyPath, InResolver);
						}
					}
					// Check to see if this is a simple soft object property (eg. not an array of soft objects).
					else if ( FSoftObjectProperty* SoftObjectProperty = CastField<FSoftObjectProperty>(Property) )
					{
						// Object boundary that can change, so we cant use the cached address safely
						InPropertyPath.SetCanSafelyUsedCachedAddress(false);

						FSoftObjectPtr SoftObject = SoftObjectProperty->GetPropertyValue_InContainer(InContainer, ArrayIndex);

						// If it's an object we need to get the value of the property in the container first before we 
						// can continue, if the object is null we safely stop processing the chain of properties.
						if ( UObject* CurrentObject = SoftObject.Get() )
						{
							InPropertyPath.SetCachedLastContainer(CurrentObject, SegmentIndex);
							return IteratePropertyPathRecursive(CurrentObject->GetClass(), CurrentObject, SegmentIndex + 1, InPropertyPath, InResolver);
						}
					}
					// Check to see if this is a simple structure (eg. not an array of structures)
					else if ( FStructProperty* StructProp = CastField<FStructProperty>(Property) )
					{
						// Recursively call back into this function with the structure property and container value
						return IteratePropertyPathRecursive<void>(StructProp->Struct, StructProp->ContainerPtrToValuePtr<void>(InContainer, ArrayIndex), SegmentIndex + 1, InPropertyPath, InResolver);
					}
					else if ( FArrayProperty* ArrayProp = CastField<FArrayProperty>(Property) )
					{
						// Dynamic array boundary that can change, so we cant use the cached address safely
						InPropertyPath.SetCanSafelyUsedCachedAddress(false);

						// It is an array, now check to see if this is an array of structures
						if ( FStructProperty* ArrayOfStructsProp = CastField<FStructProperty>(ArrayProp->Inner) )
						{
							FScriptArrayHelper_InContainer ArrayHelper(ArrayProp, InContainer);
							if ( ArrayHelper.IsValidIndex(ArrayIndex) )
							{
								// Recursively call back into this function with the array element and container value
								return IteratePropertyPathRecursive<void>(ArrayOfStructsProp->Struct, static_cast<void*>(ArrayHelper.GetRawPtr(ArrayIndex)), SegmentIndex + 1, InPropertyPath, InResolver);
							}
						}
						// if it's not an array of structs, maybe it's an array of classes
						//else if ( FObjectProperty* ObjectProperty = CastField<FObjectProperty>(ArrayProp->Inner) )
						{
							//TODO Add support for arrays of objects.
						}
					}
					else if( FSetProperty* SetProperty = CastField<FSetProperty>(Property) )
					{
						// TODO: we dont support set properties yet
					}
					else if( FMapProperty* MapProperty = CastField<FMapProperty>(Property) )
					{
						// TODO: we dont support map properties yet
					}
				}
			}
			else
			{
				// If it's the final segment, use the resolver to get the value.
				if (bFinalSegment)
				{
					return InResolver.Resolve(static_cast<ContainerType*>(InContainer), InPropertyPath);
				}
				else
				{
					// If it's not the final segment, but still a function, we're going to treat it as an Object* getter.
					// in the hopes that it leads to another object that we can resolve the next segment on.  These
					// getter functions must be very simple.

					UObject* CurrentObject = nullptr;
					FProperty* GetterProperty = nullptr;
					FInternalGetterResolver<UObject*> GetterResolver(CurrentObject, GetterProperty);

					FCachedPropertyPath TempPath(Segment);
					if (GetterResolver.Resolve(InContainer, TempPath))
					{
						if (CurrentObject)
						{
							InPropertyPath.SetCanSafelyUsedCachedAddress(false);
							InPropertyPath.SetCachedLastContainer(CurrentObject, SegmentIndex);

							return IteratePropertyPathRecursive(CurrentObject->GetClass(), CurrentObject, SegmentIndex + 1, InPropertyPath, InResolver);
						}
					}
				}
			}
		}

		return false;
	}
```




## 注册游戏设置
``` cpp
void UGameSettingRegistry::RegisterSetting(UGameSetting* InSetting)
{
	if (InSetting)
	{
		// 添加到顶层中
		TopLevelSettings.Add(InSetting);
		// 指定游戏设置的注册器
		InSetting->SetRegistry(this);
		// 内部注册绑定代理
		RegisterInnerSettings(InSetting);
	}
}
```
``` cpp

void UGameSettingRegistry::RegisterInnerSettings(UGameSetting* InSetting)
{
	// 把游戏设置的变更绑定到自己身上
	InSetting->OnSettingChangedEvent.AddUObject(this, &ThisClass::HandleSettingChanged);
	InSetting->OnSettingAppliedEvent.AddUObject(this, &ThisClass::HandleSettingApplied);
	InSetting->OnSettingEditConditionChangedEvent.AddUObject(this, &ThisClass::HandleSettingEditConditionsChanged);

	// Not a fan of this, but it makes sense to aggregate action events for simplicity.
	// 不太喜欢这种做法，但为了便于操作，将动作事件进行汇总是合理的。
	if (UGameSettingAction* ActionSetting = Cast<UGameSettingAction>(InSetting))
	{
		ActionSetting->OnExecuteNamedActionEvent.AddUObject(this, &ThisClass::HandleSettingNamedAction);
	}
	// Not a fan of this, but it makes sense to aggregate navigation events for simplicity.
	// 不太喜欢这种做法，但为了便于操作，将导航事件进行汇总是合理的。
	else if (UGameSettingCollectionPage* NewPageCollection = Cast<UGameSettingCollectionPage>(InSetting))
	{
		// 这里是从UMG点击过来的 void UGameSettingCollectionPage::ExecuteNavigation()
		// UMG 是通过反射生成的
		NewPageCollection->OnExecuteNavigationEvent.AddUObject(this, &ThisClass::HandleSettingNavigation);
	}

#if !UE_BUILD_SHIPPING
	// 避免重复注册
	ensureAlwaysMsgf(!RegisteredSettings.Contains(InSetting), TEXT("This setting has already been registered!"));
	// 名字必须唯一
	ensureAlwaysMsgf(nullptr == RegisteredSettings.FindByPredicate([&InSetting](UGameSetting* ExistingSetting) { return InSetting->GetDevName() == ExistingSetting->GetDevName(); }), TEXT("A setting with this DevName has already been registered!  DevNames must be unique within a registry."));
#endif

	RegisteredSettings.Add(InSetting);

	for (UGameSetting* ChildSetting : InSetting->GetChildSettings())
	{
		// 递归注册子设置 不再添加到TopLevelSettings中了
		RegisterInnerSettings(ChildSetting);
	}
}
```
``` cpp
	// 顶层的设置 只注册顶层的 不需要递归注册子项!
	UPROPERTY(Transient)
	TArray<TObjectPtr<UGameSetting>> TopLevelSettings;

	// 已经注册的设置
	UPROPERTY(Transient)
	TArray<TObjectPtr<UGameSetting>> RegisteredSettings;
```


## 总结
本节我们主要讲解了游戏设置注册器.但还没有讲解游戏设置这块最核心的内容!