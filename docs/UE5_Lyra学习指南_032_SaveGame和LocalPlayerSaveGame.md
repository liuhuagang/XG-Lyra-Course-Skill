# UE5_Lyra学习指南_032_SaveGame和LocalPlayerSaveGame

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_032\_SaveGame和LocalPlayerSaveGame](#ue5_lyra学习指南_032_savegame和localplayersavegame)
	- [概述](#概述)
	- [SaveGame](#savegame)
	- [LocalPlayerSaveGame](#localplayersavegame)
	- [同步读取存档](#同步读取存档)
		- [磁盘读取](#磁盘读取)
		- [加载后修正](#加载后修正)
		- [创建新存档](#创建新存档)
		- [修正(初始化)存档](#修正初始化存档)
	- [存档版本](#存档版本)
	- [异步读取存档](#异步读取存档)
	- [同步存储存档](#同步存储存档)
		- [函数入口](#函数入口)
		- [存档前修复逻辑](#存档前修复逻辑)
		- [存档后完成逻辑](#存档后完成逻辑)
	- [异步存储存档](#异步存储存档)
	- [部分辅助函数](#部分辅助函数)
		- [获取存档进程](#获取存档进程)
	- [代码](#代码)
		- [LocalPlayerSaveGame](#localplayersavegame-1)
	- [总结](#总结)



## 概述
本节主要讲解LocalPlayerSaveGame,为Lyra的共享设置做铺垫.
SaveGame我们不讲了.这个太基础了.就是存档系统
## SaveGame
引擎源码注释
``` txt
/** 
 *	This class acts as a base class for a save game object that can be used to save state about the game. 
 *	When you create your own save game subclass, you would add member variables for the information that you want to save.
 *	Then when you want to save a game, create an instance of this object using CreateSaveGameObject, fill in the data, and use SaveGameToSlot, providing a slot name.
 *	To load the game you then just use LoadGameFromSlot, and read the data from the resulting object.
 *
 *	@see https://docs.unrealengine.com/latest/INT/Gameplay/SaveGame
 */
```
``` txt

/**
* 本类作为保存游戏对象的基类，可用于保存有关游戏的状态信息。
* 当您创建自己的保存游戏子类时，您需要为想要保存的信息添加成员变量。
* 然后，当您要保存游戏时，使用 CreateSaveGameObject 创建此对象的实例，填写数据，并使用 SaveGameToSlot 将数据保存到指定的槽位，同时提供槽位名称。
* 要加载游戏，您只需使用 LoadGameFromSlot，然后从生成的对象中读取数据即可。*
*	请参阅：https://docs.unrealengine.com/latest/INT/Gameplay/SaveGame*/
```
``` cpp
UCLASS(abstract, Blueprintable, BlueprintType, MinimalAPI)
class USaveGame : public UObject
{
	/**
	 *	@see UGameplayStatics::CreateSaveGameObject
	 *	@see UGameplayStatics::SaveGameToSlot
	 *	@see UGameplayStatics::DoesSaveGameExist
	 *	@see UGameplayStatics::LoadGameFromSlot
	 *	@see UGameplayStatics::DeleteGameInSlot
	 */

	GENERATED_BODY()
};

```
如何设计一个最简单的存档系统.
首先定义一个存档的信息.
然后定义一个管理存档的信息.
现在我们有五个存档,以及一个管理存档的存档.
那么在Saved/SaveGames下的文件如下[路径根据情况可能会不同]:
```txt
ArchivePanelInformation.sav
ArchiveSlot_0.sav
ArchiveSlot_1.sav
ArchiveSlot_2.sav
ArchiveSlot_3.sav
ArchiveSlot_4.sav
```
## LocalPlayerSaveGame
引擎源码注释
``` txt
/**
 * Abstract subclass of USaveGame that provides utility functions that let you associate a Save Game object with a specific local player.
 * These objects can also be loaded using the functions on GameplayStatics, but you would need to call functions like InitializeSaveGame manually.
 * For simple games, it is fine to blueprint this class directly and add parameters and override functions in blueprint,
 * but for complicated games you will want to subclass this in native code and set up proper versioning.
 */
```


``` txt
/**
* 是 USaveGame 的一个抽象子类，它提供了实用函数，可让您将保存游戏对象与特定的本地玩家关联起来。
* 这些对象也可以通过 GameplayStatics 上的函数进行加载，但您需要手动调用诸如 InitializeSaveGame 这样的函数。
* 对于简单的游戏，可以直接在蓝图中定义这个类，并在蓝图中添加参数并重写函数即可；
* 但对于复杂的游戏，您需要在原生代码中对其进行子类化，并设置适当的版本控制。
*/
```
详细代码注释在代码章节.
以下列出存档使用流程

## 同步读取存档

### 磁盘读取
入口函数

``` cpp

	/**
	 * Synchronously loads a save game object in the specified slot for the local player, stalling the main thread until it completes.
	 * This will return null for invalid parameters, but will create a new instance if the parameters are valid but loading fails.
	 */
	/**
	 * 同步加载指定槽位中本地玩家的存档对象，会阻塞主线程直至加载完成。
	 * 对于无效参数，此方法将返回 null；但如果参数有效但加载失败，则会创建一个新的实例。
	 */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	static ENGINE_API ULocalPlayerSaveGame* LoadOrCreateSaveGameForLocalPlayer(TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, APlayerController* LocalPlayerController, const FString& SlotName);

	/**
	 * Native version of above function, this takes a ULocalPlayer because you can have a local player before a player controller.
	 * This will return null for invalid parameters, but will create a new instance if the parameters are valid but loading fails.
	 */
	/**
	 * 上述函数的原生版本，此函数接收一个 ULocalPlayer 参数，因为在有玩家控制器之前，您也可以先存在本地玩家。
	 * 对于无效参数，此函数将返回 null，但如果参数有效但加载失败，则会创建一个新的实例。
	 */
	static ENGINE_API ULocalPlayerSaveGame* LoadOrCreateSaveGameForLocalPlayer(TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, const ULocalPlayer* LocalPlayer, const FString& SlotName);


```
``` cpp
ULocalPlayerSaveGame* ULocalPlayerSaveGame::LoadOrCreateSaveGameForLocalPlayer(TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, APlayerController* LocalPlayerController, const FString& SlotName)
{
	const ULocalPlayer* LocalPlayer = LocalPlayerController ? LocalPlayerController->GetLocalPlayer() : nullptr;

	if (!ensure(LocalPlayer))
	{
		UE_LOG(LogPlayerManagement, Error, TEXT("LoadOrCreateSaveGameForLocalPlayer called with null or remote player controller!"));
		return nullptr;
	}
	// 实际调用
	return LoadOrCreateSaveGameForLocalPlayer(SaveGameClass, LocalPlayer, SlotName);
}

ULocalPlayerSaveGame* ULocalPlayerSaveGame::LoadOrCreateSaveGameForLocalPlayer(TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, const ULocalPlayer* LocalPlayer, const FString& SlotName)
{
	if (!ensure(LocalPlayer))
	{
		UE_LOG(LogPlayerManagement, Error, TEXT("LoadOrCreateSaveGame called with null local player!"));
		return nullptr;
	}

	if (!ensure(SaveGameClass))
	{
		UE_LOG(LogPlayerManagement, Error, TEXT("LoadOrCreateSaveGame called with null SaveGameClass!"));
		return nullptr;
	}

	if (!ensure(SlotName.Len() > 0))
	{
		UE_LOG(LogPlayerManagement, Error, TEXT("LoadOrCreateSaveGame called with empty slot name!"));
		return nullptr;
	}

	int32 UserIndex = LocalPlayer->GetPlatformUserIndex();

	// If the save game exists, load it.
	ULocalPlayerSaveGame* LoadedSave = nullptr;
	USaveGame* BaseSave = nullptr;
	// 如果存在就去磁盘中读取它,将其反序列化
	if (UGameplayStatics::DoesSaveGameExist(SlotName, UserIndex))
	{
		BaseSave = UGameplayStatics::LoadGameFromSlot(SlotName, UserIndex);
	}

	// 读取完成后 进行加载后处理
	LoadedSave = ProcessLoadedSave(BaseSave, SlotName, UserIndex, SaveGameClass, LocalPlayer);

	return LoadedSave;
}


```
### 加载后修正
``` cpp

/** Internal helper function used by both the sync and async save */
/** 用于同步和异步保存操作的内部辅助函数 */
static ENGINE_API ULocalPlayerSaveGame* ProcessLoadedSave(USaveGame* BaseSave, const FString& SlotName, const int32 UserIndex, TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, const ULocalPlayer* LocalPlayer);
```
``` cpp

ULocalPlayerSaveGame* ULocalPlayerSaveGame::ProcessLoadedSave(USaveGame* BaseSave, const FString& SlotName, const int32 UserIndex, TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, const ULocalPlayer* LocalPlayer)
{
	ULocalPlayerSaveGame* LoadedSave = Cast<ULocalPlayerSaveGame>(BaseSave);
	// 判断加载的存档是否正确,不正确就不使用
	if (BaseSave && (!LoadedSave || !LoadedSave->IsA(SaveGameClass.Get())))
	{
		UE_LOG(LogPlayerManagement, Warning, TEXT("ProcessLoadedSave found invalid save game object %s in slot %s for player %d!"), *GetNameSafe(LoadedSave), *SlotName, UserIndex);
		LoadedSave = nullptr;
	}

	if (LoadedSave == nullptr)
	{
		// 创建一个空白的新存档 并执行初始化存档 然后修正
		LoadedSave = CreateNewSaveGameForLocalPlayer(SaveGameClass, LocalPlayer, SlotName);
	}
	else
	{
		// 直接进行修正
		LoadedSave->InitializeSaveGame(LocalPlayer, SlotName, true);
	}

	return LoadedSave;
}
``` 

### 创建新存档
``` cpp
	/**
	 * Create a brand new save game, possibly ready for saving but without trying to load from disk. This will always succeed.
	 */
	 /** 创建一个新的存档游戏，该存档可能已准备好进行保存，但无需尝试从磁盘加载。这样做总是能够成功完成. */
	static ENGINE_API ULocalPlayerSaveGame* CreateNewSaveGameForLocalPlayer(TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, const ULocalPlayer* LocalPlayer, const FString& SlotName);
```
``` cpp
ULocalPlayerSaveGame* ULocalPlayerSaveGame::CreateNewSaveGameForLocalPlayer(TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, const ULocalPlayer* LocalPlayer, const FString& SlotName)
{
	ULocalPlayerSaveGame* LoadedSave = Cast<ULocalPlayerSaveGame>(UGameplayStatics::CreateSaveGameObject(SaveGameClass));
	if (ensure(LoadedSave))
	{
		LoadedSave->InitializeSaveGame(LocalPlayer, SlotName, false);
	}
	return LoadedSave;
}	

```

### 修正(初始化)存档
``` cpp
	/** Initializes this save after either loading or initial creation, automatically called by load/create functions above */
	/** 在加载或初始创建后初始化此保存功能，上述的加载/创建函数会自动调用此操作 */
	ENGINE_API virtual void InitializeSaveGame(const ULocalPlayer* LocalPlayer, FString InSlotName, bool bWasLoaded);


```
``` cpp
void ULocalPlayerSaveGame::InitializeSaveGame(const ULocalPlayer* LocalPlayer, FString InSlotName, bool bWasLoaded)
{
	SetLocalPlayer(LocalPlayer);
	SetSaveSlotName(InSlotName);

	if (bWasLoaded)
	{
		LoadedDataVersion = SavedDataVersion;
		HandlePostLoad();
	}
	else
	{
		ResetToDefault();
	}
}

```
## 存档版本
``` cpp
void ULocalPlayerSaveGame::ResetToDefault()
{
	SavedDataVersion = GetInvalidDataVersion();
	LoadedDataVersion = SavedDataVersion;

	OnResetToDefault();
}
int32 ULocalPlayerSaveGame::GetInvalidDataVersion() const
{
	return -1;
}
```
新创建的存档为-1,该类的版本为0, 子类可以为1,2,3,4等等

``` cpp
	/** 
	 * The value of GetLatestDataVersion when this was last saved.
	 * Subclasses can override GetLatestDataVersion and then handle fixups in HandlePostLoad.
	 * This defaults to 0 so old save games that didn't previously subclass ULocalPlayerSaveGame will have 0 instead of the invalid version.
	 */

	/**
	 * 此数据上次保存时的“获取最新数据版本”值。
	 * 子类可以重写“获取最新数据版本”方法，然后在“处理加载后操作”中进行修正处理。
	 * 此值默认为 0，因此那些之前未继承自 ULocalPlayerSaveGame 的旧保存游戏将显示为 0 而非无效版本。
	 */
	UPROPERTY()
	int32 SavedDataVersion = 0;

	/**
	 * The value of SavedDataVersion when a save was last loaded, this will be -1 for newly created saves
	 */
	 /** “SavedDataVersion”的值表示上次加载保存文件时的版本号，对于新创建的保存文件，其值将为 -1 。*/
	UPROPERTY(Transient)
	int32 LoadedDataVersion = -1;

```
## 异步读取存档
使用流程基本和同步相似.
区别在于需要通过Lambda传递一个加载完成后的执行代理.
同样会执行修正存档的过程.
``` cpp
bool ULocalPlayerSaveGame::AsyncLoadOrCreateSaveGameForLocalPlayer(TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, APlayerController* LocalPlayerController, const FString& SlotName, FOnLocalPlayerSaveGameLoaded Delegate)
{
	const ULocalPlayer* LocalPlayer = LocalPlayerController ? LocalPlayerController->GetLocalPlayer() : nullptr;

	if (!ensure(LocalPlayer))
	{
		UE_LOG(LogPlayerManagement, Error, TEXT("AsyncLoadOrCreateSaveGameForLocalPlayer called with null or remote player controller!"));
		return false;
	}

	// Simple lambda that calls the dynamic delegate
	FOnLocalPlayerSaveGameLoadedNative Lambda = FOnLocalPlayerSaveGameLoadedNative::CreateLambda([Delegate]
		(ULocalPlayerSaveGame* LoadedSave)
		{
			Delegate.ExecuteIfBound(LoadedSave);
		});
	// 调用实际内部版本
	return AsyncLoadOrCreateSaveGameForLocalPlayer(SaveGameClass, LocalPlayer, SlotName, Lambda);
}

bool ULocalPlayerSaveGame::AsyncLoadOrCreateSaveGameForLocalPlayer(TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, const ULocalPlayer* LocalPlayer, const FString& SlotName, FOnLocalPlayerSaveGameLoadedNative Delegate)
{
	if (!ensure(LocalPlayer))
	{
		UE_LOG(LogPlayerManagement, Error, TEXT("AsyncLoadOrCreateSaveGame called with null local player!"));
		return false;
	}

	if (!ensure(SaveGameClass))
	{
		UE_LOG(LogPlayerManagement, Error, TEXT("AsyncLoadOrCreateSaveGame called with null SaveGameClass!"));
		return false;
	}

	if (!ensure(SlotName.Len() > 0))
	{
		UE_LOG(LogPlayerManagement, Error, TEXT("AsyncLoadOrCreateSaveGame called with empty slot name!"));
		return false;
	}
	
	FAsyncLoadGameFromSlotDelegate Lambda = FAsyncLoadGameFromSlotDelegate::CreateWeakLambda(LocalPlayer, [LocalPlayer, Delegate, SaveGameClass]
		(const FString& SlotName, const int32 UserIndex, USaveGame* BaseSave) 
		{
			ULocalPlayerSaveGame* LoadedSave = ProcessLoadedSave(BaseSave, SlotName, UserIndex, SaveGameClass, LocalPlayer);
			
			Delegate.ExecuteIfBound(LoadedSave);
		});
	// 调用引擎底层 加载完毕后执行Lambda的修正函数
	UGameplayStatics::AsyncLoadGameFromSlot(SlotName, LocalPlayer->GetPlatformUserIndex(), Lambda);

	return true;
}

```

## 同步存储存档
### 函数入口
``` cpp
	/**
	 * Synchronously save using the slot and user index, stalling the main thread until it completes. 
	 * This will return true if the save was requested, and errors should be handled by the HandlePostSave function that will be called immediately.
	 */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual bool SaveGameToSlotForLocalPlayer();
```
``` cpp

bool ULocalPlayerSaveGame::SaveGameToSlotForLocalPlayer()
{
	// Subclasses can override this to add additional checks like if one is in progress
	// 子类可以重写此方法来添加额外的检查，例如在某个操作正在进行时进行检查。
	if (!ensure(GetLocalPlayer()))
	{
		UE_LOG(LogPlayerManagement, Error, TEXT("SaveGameToSlotForLocalPlayer called with null local player!"));
		return false;
	}
	// 获取平台用于索引
	int32 RequestUserIndex = GetPlatformUserIndex();
	// 获取存档名称
	FString RequestSlotName = GetSaveSlotName();
	if (!ensure(RequestSlotName.Len() > 0))
	{
		UE_LOG(LogPlayerManagement, Error, TEXT("SaveGameToSlotForLocalPlayer called with empty slot name!"));
		return false;
	}
	// 存档前修复逻辑
	HandlePreSave();

	bool bSuccess = UGameplayStatics::SaveGameToSlot(this, RequestSlotName, RequestUserIndex);

	ProcessSaveComplete(RequestSlotName, RequestUserIndex, bSuccess, CurrentSaveRequest);

	return true;
}
```
### 存档前修复逻辑

``` cpp
void ULocalPlayerSaveGame::HandlePreSave()
{
	// 蓝图事件
	OnPreSave();

	// Set the save data version and increment the requested count

	SavedDataVersion = GetLatestDataVersion();
	CurrentSaveRequest++;

	UE_LOG(LogPlayerManagement, Verbose, TEXT("Starting to save game %s request %d to slot %s for user %d"), *GetName(), CurrentSaveRequest, *GetSaveSlotName(), GetPlatformUserIndex());
}

int32 ULocalPlayerSaveGame::GetLatestDataVersion() const
{
	// Override with game-specific logic
	return 0;
}
```
### 存档后完成逻辑
``` cpp
	/** Internal callback that will call HandlePostSave */
	/** 内部回调函数，该函数将调用“HandlePostSave”函数 */
	ENGINE_API virtual void ProcessSaveComplete(const FString& SlotName, const int32 UserIndex, bool bSuccess, int32 SaveRequest);

```
``` cpp
void ULocalPlayerSaveGame::ProcessSaveComplete(const FString& SlotName, const int32 UserIndex, bool bSuccess, int32 SaveRequest)
{
	// Now that a save completed, update the success/failure
	// Every time CurrentSaveRequest is incremented it will result in setting either success or error, but there could be multiple in flight that are processed in order
	// 现在保存操作已完成，需更新成功/失败的状态
	// 每次调用 CurrentSaveRequest 的值增加时，都会决定是设置“成功”还是“失败”状态，但可能存在多个正在进行中的操作，它们会按照顺序进行处理
	if (bSuccess)
	{
		ensure(CurrentSaveRequest > LastSuccessfulSaveRequest);
		LastSuccessfulSaveRequest = CurrentSaveRequest;
	}
	else
	{
		ensure(CurrentSaveRequest > LastErrorSaveRequest);
		LastErrorSaveRequest = CurrentSaveRequest;
	}

	HandlePostSave(bSuccess);
}

```
``` cpp
void ULocalPlayerSaveGame::HandlePostSave(bool bSuccess)
{
	// Override for error handling
	if (bSuccess)
	{
		UE_LOG(LogPlayerManagement, Verbose, TEXT("Successfully saved game %s request %d to slot %s for user %d"), *GetName(), LastSuccessfulSaveRequest, *GetSaveSlotName(), GetPlatformUserIndex());
	}
	else
	{
		UE_LOG(LogPlayerManagement, Error, TEXT("Failed to save game %s request %d to slot %s for user %d!"), *GetName(), LastErrorSaveRequest, *GetSaveSlotName(), GetPlatformUserIndex());
	}
	// 蓝图事件
	OnPostSave(bSuccess);
}


```

## 异步存储存档
原理和同步一样.只是执行后逻辑需要通过Lambda传递
``` cpp
	/**
	 * Asynchronously save to the slot and user index.
	 * This will return true if the save was requested, and errors should be handled by the HandlePostSave function after the save succeeds or fails
	 */
	/**
	 * 异步将数据保存至存储槽和用户索引中。
	 * 如果保存操作被请求，此函数将返回真。保存成功或失败后，错误应由 HandlePostSave 函数进行处理。
	 */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual bool AsyncSaveGameToSlotForLocalPlayer();

```
``` cpp

bool ULocalPlayerSaveGame::AsyncSaveGameToSlotForLocalPlayer()
{
	// Subclasses can override this to add additional checks like if one is in progress
	// 子类可以重写此方法来添加额外的检查，例如在某个操作正在进行时进行检查。
	if (!ensure(GetLocalPlayer()))
	{
		UE_LOG(LogPlayerManagement, Error, TEXT("AsyncSaveGameToSlotForLocalPlayer called with null local player!"));
		return false;
	}
	
	int32 RequestUserIndex = GetPlatformUserIndex();
	FString RequestSlotName = GetSaveSlotName();
	if (!ensure(RequestSlotName.Len() > 0))
	{
		UE_LOG(LogPlayerManagement, Error, TEXT("AsyncSaveGameToSlotForLocalPlayer called with empty slot name!"));
		return false;
	}

	// 存档前修复逻辑
	HandlePreSave();

	// Start an async save and pass through the current save request count
	// 启动异步保存操作，并传递当前的保存请求次数
	FAsyncSaveGameToSlotDelegate SavedDelegate;
	SavedDelegate.BindUObject(this, &ULocalPlayerSaveGame::ProcessSaveComplete, CurrentSaveRequest);
	UGameplayStatics::AsyncSaveGameToSlot(this, RequestSlotName, RequestUserIndex, SavedDelegate);

	return true;
}
```
## 部分辅助函数
### 获取存档进程
``` cpp
// 是否从已有的存档中加载
bool ULocalPlayerSaveGame::WasLoaded() const
{
	return LoadedDataVersion != GetInvalidDataVersion();
}
// 是否正在存档
bool ULocalPlayerSaveGame::IsSaveInProgress() const
{
	// 是否有存档在
	return (CurrentSaveRequest > LastErrorSaveRequest) || (CurrentSaveRequest > LastSuccessfulSaveRequest);
}
// 最近一次存档是否成功
bool ULocalPlayerSaveGame::WasLastSaveSuccessful() const
{
	return (WasSaveRequested() && LastSuccessfulSaveRequest > LastErrorSaveRequest);
}
// 是否以及存过档,可能在进程中
bool ULocalPlayerSaveGame::WasSaveRequested() const
{
	return CurrentSaveRequest > 0;
}

```



## 代码
### LocalPlayerSaveGame
``` cpp

/**
 * Abstract subclass of USaveGame that provides utility functions that let you associate a Save Game object with a specific local player.
 * These objects can also be loaded using the functions on GameplayStatics, but you would need to call functions like InitializeSaveGame manually.
 * For simple games, it is fine to blueprint this class directly and add parameters and override functions in blueprint,
 * but for complicated games you will want to subclass this in native code and set up proper versioning.
 */
 /**
* 是 USaveGame 的一个抽象子类，它提供了实用函数，可让您将保存游戏对象与特定的本地玩家关联起来。
* 这些对象也可以通过 GameplayStatics 上的函数进行加载，但您需要手动调用诸如 InitializeSaveGame 这样的函数。
* 对于简单的游戏，可以直接在蓝图中定义这个类，并在蓝图中添加参数并重写函数即可；
* 但对于复杂的游戏，您需要在原生代码中对其进行子类化，并设置适当的版本控制。
*/
UCLASS(abstract, Blueprintable, BlueprintType, MinimalAPI)
class ULocalPlayerSaveGame : public USaveGame
{
	GENERATED_BODY()
public:

	/**
	 * Synchronously loads a save game object in the specified slot for the local player, stalling the main thread until it completes.
	 * This will return null for invalid parameters, but will create a new instance if the parameters are valid but loading fails.
	 */
	/**
	 * 同步加载指定槽位中本地玩家的存档对象，会阻塞主线程直至加载完成。
	 * 对于无效参数，此方法将返回 null；但如果参数有效但加载失败，则会创建一个新的实例。
	 */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	static ENGINE_API ULocalPlayerSaveGame* LoadOrCreateSaveGameForLocalPlayer(TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, APlayerController* LocalPlayerController, const FString& SlotName);

	/**
	 * Native version of above function, this takes a ULocalPlayer because you can have a local player before a player controller.
	 * This will return null for invalid parameters, but will create a new instance if the parameters are valid but loading fails.
	 */
	/**
	 * 上述函数的原生版本，此函数接收一个 ULocalPlayer 参数，因为在有玩家控制器之前，您也可以先存在本地玩家。
	 * 对于无效参数，此函数将返回 null，但如果参数有效但加载失败，则会创建一个新的实例。
	 */
	static ENGINE_API ULocalPlayerSaveGame* LoadOrCreateSaveGameForLocalPlayer(TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, const ULocalPlayer* LocalPlayer, const FString& SlotName);

	/**
	 * Asynchronously loads a save game object in the specified slot for the local player, if this returns true the delegate will get called later.
	 * False means the load was never scheduled, otherwise it will create and initialize a new instance before calling the delegate if loading failed.
	 */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	static ENGINE_API bool AsyncLoadOrCreateSaveGameForLocalPlayer(TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, APlayerController* LocalPlayerController, const FString& SlotName, FOnLocalPlayerSaveGameLoaded Delegate);

	/**
	 * Native version of above function, this takes a ULocalPlayer and calls a native delegate.
	 * False means the load was never scheduled, otherwise it will create and initialize a new instance before calling the delegate if loading failed.
	 */
	static ENGINE_API bool AsyncLoadOrCreateSaveGameForLocalPlayer(TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, const ULocalPlayer* LocalPlayer, const FString& SlotName, FOnLocalPlayerSaveGameLoadedNative Delegate);

	/**
	 * Create a brand new save game, possibly ready for saving but without trying to load from disk. This will always succeed.
	 */
	 /**
	  * 创建一个新的存档游戏，该存档可能已准备好进行保存，但无需尝试从磁盘加载。这样做总是能够成功完成。
	  */
	static ENGINE_API ULocalPlayerSaveGame* CreateNewSaveGameForLocalPlayer(TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, const ULocalPlayer* LocalPlayer, const FString& SlotName);

	/**
	 * Synchronously save using the slot and user index, stalling the main thread until it completes. 
	 * This will return true if the save was requested, and errors should be handled by the HandlePostSave function that will be called immediately.
	 */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual bool SaveGameToSlotForLocalPlayer();

	/**
	 * Asynchronously save to the slot and user index.
	 * This will return true if the save was requested, and errors should be handled by the HandlePostSave function after the save succeeds or fails
	 */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual bool AsyncSaveGameToSlotForLocalPlayer();


	/** Returns the local player controller this is associated with, this will be valid if it is ready to save */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual APlayerController* GetLocalPlayerController() const;

	/** Returns the local player this is associated with, this will be valid if it is ready to save */
	ENGINE_API virtual const ULocalPlayer* GetLocalPlayer() const;

	/** Associates this save game with a local player, this is called automatically during load/create */
	/** 将此存档游戏与本地玩家关联起来，此操作会在加载/创建过程中自动执行 */
	ENGINE_API virtual void SetLocalPlayer(const ULocalPlayer* LocalPlayer);

	/** Returns the platform user to save to, based on Local Player by default */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual FPlatformUserId GetPlatformUserId() const;

	/** Returns the user index to save to, based on Local Player by default */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual int32 GetPlatformUserIndex() const;

	/** Returns the save slot name to use */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual FString GetSaveSlotName() const;

	/** Sets the save slot name for any future saves */
	/** 为未来的所有保存操作设置保存槽的名称 */
	ENGINE_API virtual void SetSaveSlotName(const FString& SlotName);

	/** Returns the game-specific version number this was last saved/loaded as */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual int32 GetSavedDataVersion() const;

	/** Returns the invalid save data version, which means it has never been saved/loaded */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual int32 GetInvalidDataVersion() const;

	/** Returns the latest save data version, this is used when the new data is saved */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual int32 GetLatestDataVersion() const;

	/** Returns true if this was loaded from an existing save */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual bool WasLoaded() const;

	/** Returns true if a save is in progress */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual bool IsSaveInProgress() const;

	/** Returns true if it has been saved at least once and the last save was successful */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual bool WasLastSaveSuccessful() const;

	/** Returns true if a save was ever requested, may still be in progress */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual bool WasSaveRequested() const;


	/** Initializes this save after either loading or initial creation, automatically called by load/create functions above */
	/** 在加载或初始创建后初始化此保存功能，上述的加载/创建函数会自动调用此操作 */
	ENGINE_API virtual void InitializeSaveGame(const ULocalPlayer* LocalPlayer, FString InSlotName, bool bWasLoaded);

	/** Resets all saved data to default values, called when the load fails or manually */
	/** 将所有保存的数据重置为默认值，该操作在加载失败时或手动执行时会触发 */
	UFUNCTION(BlueprintCallable, Category = "SaveGame|LocalPlayer")
	ENGINE_API virtual void ResetToDefault();

	/** Blueprint event called to reset all saved data to default, called when the load fails or manually */
	/** 一个蓝图事件，用于将所有保存的数据重置为默认值，该事件在加载失败时或手动调用时触发 */
	UFUNCTION(BlueprintImplementableEvent, Category = "SaveGame|LocalPlayer")
	ENGINE_API void OnResetToDefault();

	/** Called after loading, this is not called for newly created saves */
	/** 在加载完成后会调用此函数，对于新创建的保存文件则不会调用此函数 */
	ENGINE_API virtual void HandlePostLoad();

	/** Blueprint event called after loading, is not called for newly created saves */
	/** 在加载完成后触发的蓝图事件，对于新创建的存档不会触发此事件 */
	UFUNCTION(BlueprintImplementableEvent, Category = "SaveGame|LocalPlayer")
	ENGINE_API void OnPostLoad();

	/** Called before saving, do any game-specific fixup here */
	ENGINE_API virtual void HandlePreSave();

	/** Blueprint event called before saving, do any game-specific fixup here  */
	UFUNCTION(BlueprintImplementableEvent, Category = "SaveGame|LocalPlayer")
	ENGINE_API void OnPreSave();

	/** Called after saving finishes with success/failure result */
	ENGINE_API virtual void HandlePostSave(bool bSuccess);

	/** Blueprint event called after saving finishes with success/failure result */
	UFUNCTION(BlueprintImplementableEvent, Category = "SaveGame|LocalPlayer")
	ENGINE_API void OnPostSave(bool bSuccess);

protected:
	/** Internal callback that will call HandlePostSave */
	ENGINE_API virtual void ProcessSaveComplete(const FString& SlotName, const int32 UserIndex, bool bSuccess, int32 SaveRequest);

	/** Internal helper function used by both the sync and async save */
	/** 用于同步和异步保存操作的内部辅助函数 */
	static ENGINE_API ULocalPlayerSaveGame* ProcessLoadedSave(USaveGame* BaseSave, const FString& SlotName, const int32 UserIndex, TSubclassOf<ULocalPlayerSaveGame> SaveGameClass, const ULocalPlayer* LocalPlayer);

	/** The local player this is connected to, can be null if subclasses override Get/Set Local Player or it hasn't been initialized */
	UPROPERTY(Transient)
	TObjectPtr<const ULocalPlayer> OwningPlayer;

	/** The slot name this was loaded from and that will be used to save to in the future */
	UPROPERTY(Transient)
	FString SaveSlotName;

	/** 
	 * The value of GetLatestDataVersion when this was last saved.
	 * Subclasses can override GetLatestDataVersion and then handle fixups in HandlePostLoad.
	 * This defaults to 0 so old save games that didn't previously subclass ULocalPlayerSaveGame will have 0 instead of the invalid version.
	 */

	/**
	 * 此数据上次保存时的“获取最新数据版本”值。
	 * 子类可以重写“获取最新数据版本”方法，然后在“处理加载后操作”中进行修正处理。
	 * 此值默认为 0，因此那些之前未继承自 ULocalPlayerSaveGame 的旧保存游戏将显示为 0 而非无效版本。
	 */
	UPROPERTY()
	int32 SavedDataVersion = 0;

	/**
	 * The value of SavedDataVersion when a save was last loaded, this will be -1 for newly created saves
	 */
	 /** “SavedDataVersion”的值表示上次加载保存文件时的版本号，对于新创建的保存文件，其值将为 -1 。*/
	UPROPERTY(Transient)
	int32 LoadedDataVersion = -1;


	/** Integer that is incremented every time a save has been requested in the current session, can be used to know if one is in progress */
	UPROPERTY(Transient)
	int32 CurrentSaveRequest = 0;

	/** Integer that is set when a save completes successfully, if this equals RequestedSaveCount then the last save was successful */
	UPROPERTY(Transient)
	int32 LastSuccessfulSaveRequest = 0;

	/** Integer that is set when a save fails */
	UPROPERTY(Transient)
	int32 LastErrorSaveRequest = 0;

};


```

## 总结
相比一个最简单的SaveGame,LocalSaveGame 添加了版本验证,存档进程查询等等功能.同时支持我们去重写相关虚函数完成存档前后的验证修复!