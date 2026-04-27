# UE5_Lyra学习指南_104_异步混入AsyncMixin

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_104\_异步混入AsyncMixin](#ue5_lyra学习指南_104_异步混入asyncmixin)
	- [概述](#概述)
	- [设计思路](#设计思路)
	- [使用流程](#使用流程)
	- [FNoncopyable](#fnoncopyable)
	- [FAsyncMixin](#fasyncmixin)
	- [构造函数](#构造函数)
	- [子类接收拓展](#子类接收拓展)
	- [外部调用](#外部调用)
		- [异步加载资产](#异步加载资产)
		- [异步加载主要资产](#异步加载主要资产)
		- [异步调用条件执行事件](#异步调用条件执行事件)
		- [异步调用执行事件](#异步调用执行事件)
		- [加载控制](#加载控制)
			- [启动异步加载](#启动异步加载)
			- [取消异步加载](#取消异步加载)
			- [是否正在加载](#是否正在加载)
		- [转发到内部处理](#转发到内部处理)
	- [工具函数](#工具函数)
	- [FLoadingState](#floadingstate)
	- [FAsyncStep](#fasyncstep)
	- [FAsyncScope](#fasyncscope)
	- [FAsyncCondition](#fasynccondition)
	- [调用流程](#调用流程)
		- [异步加载任务](#异步加载任务)
			- [函数入口](#函数入口)
			- [获取加载状态](#获取加载状态)
			- [转发](#转发)
			- [创建异步加载任务](#创建异步加载任务)
			- [确保下一帧能启动](#确保下一帧能启动)
			- [取消之前安排的回收操作](#取消之前安排的回收操作)
			- [轮询是否加载完成](#轮询是否加载完成)
			- [加载完成执行回调](#加载完成执行回调)
			- [绑定处理尝试完成函数的重入](#绑定处理尝试完成函数的重入)
				- [资产的异步加载回调](#资产的异步加载回调)
				- [用户自己定义的条件](#用户自己定义的条件)
			- [该任务执行完成回调](#该任务执行完成回调)
			- [回收资源](#回收资源)
			- [内存回收](#内存回收)
		- [异步加载主要资产](#异步加载主要资产-1)
		- [异步条件执行](#异步条件执行)
		- [异步执行](#异步执行)
	- [总结](#总结)



## 概述
本文主要讲解Lyra项目中为了简化资产异步加载后回调所使用的AsyncMixin插件代码
## 设计思路
``` cpp
//TODO I think we need to introduce a retention policy, preloads automatically stay in memory until canceled
//     but what if you want to preload individual items just using the AsyncLoad functions?  I don't want to
//     introduce individual policies per call, or introduce a whole set of preload vs asyncloads, so would
//     would rather have a retention policy.  Should it be a member and actually create real memory when
//     you inherit from AsyncMixin, or should it be a template argument?
```

```
//  有待完成 我认为我们需要制定一个保留策略，即自动预加载的内容会一直保留在内存中直至被取消。
//  但如果您只是想通过 AsyncLoad 函数来预加载单个项目呢？我不想
//  对于每次调用，可以单独设定具体的策略，或者引入一套预加载与异步加载的组合方案，因此
//  我更倾向于采用保留策略。它应该是一个成员变量，并且在继承自 AsyncMixin 时实际创建实际的内存，还是
//  应该是一个模板参数呢？
```
``` cpp
//enum class EAsyncMixinRetentionPolicy : uint8
//{
//	Default,
//	KeepResidentUntilComplete,
//	KeepResidentUntilCancel
//};

```
## 使用流程
``` cpp

/**
 * The FAsyncMixin allows easier management of async loading requests, to ensure linear request handling, to make 
 * writing code much easier.  The usage pattern is as follows,
 *
 * First - inherit from FAsyncMixin, even if you're a UObject, you can also inherit from FAsyncMixin.
 *
 * Then - you can make your async loads as follows.
 * 
 * CancelAsyncLoading();			// Some objects get reused like in lists, so it's important to cancel anything you had pending doesn't complete.
 * AsyncLoad(ItemOne, CallbackOne);
 * AsyncLoad(ItemTwo, CallbackTwo);
 * StartAsyncLoading();
 * 
 * You can also include the 'this' scope safely, one of the benefits of the mix-in, is that none of the callbacks
 * are ever out of scope of the host AsyncMixin derived object.
 * e.g.
 * AsyncLoad(SomeSoftObjectPtr, [this, ...]() {
 *    
 * });
 * 
 *
 * What will happen is first we cancel any existing one(s), e.g. perhaps we are a widget that just got told to represent
 * some new thing.  What will happen is we'll Load ItemOne and ItemTwo, *THEN* we'll call the callbacks in the order you
 * requested the async loads - even if ItemOne or ItemTwo was already loaded when you request it.
 *
 * When all the async loading requests complete, OnFinishedLoading will be called.
 * 
 * If you forget to call StartAsyncLoading(), we'll call it next frame, but you should remember to call it
 * when you're done with your setup, as maybe everything is already loaded, and it will avoid a single frame
 * of a loading indicator flash, which is annoying.
 * 
 * NOTE: The FAsyncMixin also makes it safe to pass [this] as a captured input into your lambda, because it handles 
 * unhooking everything if either your owner class is destroyed, or you cancel everything.
 *
 * NOTE: FAsyncMixin doesn't add any additional memory to your class.  Several classes currently handling async loading 
 * internally allocate TSharedPtr<FStreamableHandle> members and tend to hold onto SoftObjectPaths temporary state.  The 
 * FAsyncMixin does all of this internally with a static TMap so that all of the async request memory is stored temporarily
 * and sparsely.
 * 
 * NOTE: For debugging and understanding what's going on, you should add -LogCmds="LogAsyncMixin Verbose" to the command line.
 */

```

``` txt

/**
* FAsyncMixin 使得异步加载请求的管理更加简便，从而确保请求处理的线性性，并使代码编写变得更加容易。
* 其使用模式如下：*
* 首先 - 请从 FAsyncMixin 类继承，即便您所在的类是 UObject 类型，您也可以从 FAsyncMixin 类继承。*
* 那么——您可以按照以下方式来进行异步加载。*
* 取消异步加载操作；		// 有些对象会像在列表中那样被重复使用，因此务必取消任何尚未完成的进程。
* 异步加载（ItemOne，回调函数One）；
* 异步加载（ItemTwo，回调函数Two）；
* 启动异步加载操作；*
您还可以安全地包含“this”作用域，这是混入机制的一个优点，即所有的回调函数都不会脱离宿主的 AsyncMixin 继承对象的作用域。
例如：
AsyncLoad(SomeSoftObjectPtr, [this, ...]() {*
} ；*
*
接下来会发生的情况是，首先我们会取消现有的任何操作（比如，也许我们原本是负责展示某种新内容的组件，但后来被告知要改做别的事情）。接下来我们会加载“ItemOne”和“ItemTwo”这两个项目，然后按照您所指定的顺序调用异步加载的回调函数——即便在您请求加载时，ItemOne 或 ItemTwo 已经已经被加载过了。*
当所有的异步加载请求完成后，就会调用“OnFinishedLoading”函数。*
* 如果您忘记调用 StartAsyncLoading() 方法，我们会在下一帧中调用它，但您在完成设置后仍应记得调用它。
* 因为也许此时所有内容都已经加载完成，这样就能避免出现一帧的加载指示器闪烁现象，这会让人感到很烦人。*
* 注意：FAsyncMixin 还确保了将 [this] 作为捕获输入传递给 lambda 时是安全的，因为它会处理在您的父类被销毁或者您取消所有操作的情况下解除所有关联的情况。*
* 注意：FAsyncMixin 不会为您的类增加任何额外的内存。 目前有多个类在内部处理异步加载操作，
* 它们会为成员分配 TSharedPtr<FStreamableHandle> 类型的内存，并且往往会保存临时的 SoftObjectPaths 状态。最终的译文：这个
* FAsyncMixin 在内部通过一个静态的 TMap 来完成所有这些操作，这样所有的异步请求内存都会临时且稀疏地存储起来。*
* 注意：为了进行调试并了解当前情况，您应在命令行中添加“-LogCmds="LogAsyncMixin Verbose"”这一参数。*/

```


## FNoncopyable
``` cpp
用于不能被复制的类的通用模板。从这个类派生出您的类，即可使您的类无法被复制。
```
``` cpp
class FNoncopyable
{
protected:
	// ensure the class cannot be constructed directly
	// 确保该类无法被直接实例化
	FNoncopyable() {}
	// the class should not be used polymorphically
	// 该类不应采用多态方式使用
	~FNoncopyable() {}
private:
	FNoncopyable(const FNoncopyable&);
	FNoncopyable& operator=(const FNoncopyable&);
};

```
## FAsyncMixin
## 构造函数
``` cpp
class FAsyncMixin : public FNoncopyable
{
protected:
	UE_API FAsyncMixin();

public:
	UE_API virtual ~FAsyncMixin();
}
```
``` cpp
FAsyncMixin::~FAsyncMixin()
{
	check(IsInGameThread());

	// Removing the loading state will cancel any pending loadings it was 
	// monitoring, and shouldn't receive any future callbacks for completion.
	// 移除加载状态将取消其所监控的任何正在进行的加载操作，并且此后不应再接收任何关于完成状态的回调通知。
	Loading.Remove(this);
}
```
清理回收资源
``` cpp

private:
	static UE_API TMap<FAsyncMixin*, TSharedRef<FLoadingState>> Loading;

```
## 子类接收拓展
``` cpp
protected:
	/** Called when loading starts. */
	/** 当加载开始时被调用。*/
	virtual void OnStartedLoading()
	{
	}

	/** Called when all loading has finished. */
	/** 当所有加载操作完成时会调用此方法。*/
	virtual void OnFinishedLoading()
	{
	}
```
## 外部调用
### 异步加载资产
``` cpp

protected:
	/** Async load a TSoftClassPtr<T>, call the Callback when complete. */
	/** 异步加载一个 TSoftClassPtr<T> 类型的对象，并在加载完成时调用回调函数。*/
	template <typename T = UObject>
	void AsyncLoad(TSoftClassPtr<T> SoftClass, TFunction<void()>&& Callback)
	{
		AsyncLoad(SoftClass.ToSoftObjectPath(), FSimpleDelegate::CreateLambda(MoveTemp(Callback)));
	}

	/** Async load a TSoftClassPtr<T>, call the Callback when complete. */
	/** 异步加载一个 TSoftClassPtr<T> 类型的对象，并在加载完成时调用回调函数。*/
	template <typename T = UObject>
	void AsyncLoad(TSoftClassPtr<T> SoftClass, TFunction<void(TSubclassOf<T>)>&& Callback)
	{
		AsyncLoad(SoftClass.ToSoftObjectPath(),
		          FSimpleDelegate::CreateLambda([SoftClass, UserCallback = MoveTemp(Callback)]() mutable
		          {
			          UserCallback(SoftClass.Get());
		          })
		);
	}

	/** Async load a TSoftClassPtr<T>, call the Callback when complete. */
	/** 异步加载一个 TSoftClassPtr<T> 类型的对象，并在加载完成时调用回调函数。*/
	template <typename T = UObject>
	void AsyncLoad(TSoftClassPtr<T> SoftClass, const FSimpleDelegate& Callback = FSimpleDelegate())
	{
		AsyncLoad(SoftClass.ToSoftObjectPath(), Callback);
	}

	/** Async load a TSoftObjectPtr<T>, call the Callback when complete. */
	/** 异步加载一个 TSoftClassPtr<T> 类型的对象，并在加载完成时调用回调函数。*/
	template <typename T = UObject>
	void AsyncLoad(TSoftObjectPtr<T> SoftObject, TFunction<void()>&& Callback)
	{
		AsyncLoad(SoftObject.ToSoftObjectPath(), FSimpleDelegate::CreateLambda(MoveTemp(Callback)));
	}

	/** Async load a TSoftObjectPtr<T>, call the Callback when complete. */
	/** 异步加载一个 TSoftObjectPtr<T>，加载完成后调用回调函数。*/
	template <typename T = UObject>
	void AsyncLoad(TSoftObjectPtr<T> SoftObject, TFunction<void(T*)>&& Callback)
	{
		AsyncLoad(SoftObject.ToSoftObjectPath(),
		          FSimpleDelegate::CreateLambda([SoftObject, UserCallback = MoveTemp(Callback)]() mutable
		          {
			          UserCallback(SoftObject.Get());
		          })
		);
	}

	/** Async load a TSoftObjectPtr<T>, call the Callback when complete. */
	/** 异步加载一个 TSoftObjectPtr<T>，加载完成后调用回调函数。*/
	template <typename T = UObject>
	void AsyncLoad(TSoftObjectPtr<T> SoftObject, const FSimpleDelegate& Callback = FSimpleDelegate())
	{
		AsyncLoad(SoftObject.ToSoftObjectPath(), Callback);
	}


	/** Async load an array of FSoftObjectPath, call the Callback when complete. */
	/** 异步加载一个包含 FSoftObjectPath 的数组，并在完成时调用回调函数。*/
	void AsyncLoad(const TArray<FSoftObjectPath>& SoftObjectPaths, TFunction<void()>&& Callback)
	{
		AsyncLoad(SoftObjectPaths, FSimpleDelegate::CreateLambda(MoveTemp(Callback)));
	}
```

```cpp

	/** Async load a FSoftObjectPath, call the Callback when complete. */
	/** 异步加载一个 FSoftObjectPath，加载完成后调用回调函数。*/
	UE_API void AsyncLoad(FSoftObjectPath SoftObjectPath, const FSimpleDelegate& Callback = FSimpleDelegate());


	/** Async load an array of FSoftObjectPath, call the Callback when complete. */
	/** 异步加载一个包含 FSoftObjectPath 的数组，并在完成时调用回调函数。*/
	UE_API void AsyncLoad(const TArray<FSoftObjectPath>& SoftObjectPaths,
	                      const FSimpleDelegate& Callback = FSimpleDelegate());

```
### 异步加载主要资产
``` cpp
	/** Given an array of primary assets, it loads all of the bundles referenced by properties of these assets specified in the LoadBundles array. */
	/** 给定一组主要资产，它会加载这些资产的属性中所指定的“加载资源包”数组所引用的所有资源包。*/
	template <typename T = UPrimaryDataAsset>
	void AsyncPreloadPrimaryAssetsAndBundles(const TArray<T*>& Assets, const TArray<FName>& LoadBundles,
	                                         const FSimpleDelegate& Callback = FSimpleDelegate())
	{
		TArray<FPrimaryAssetId> PrimaryAssetIds;
		for (const T* Item : Assets)
		{
			PrimaryAssetIds.Add(Item);
		}

		AsyncPreloadPrimaryAssetsAndBundles(PrimaryAssetIds, LoadBundles, Callback);
	}

	/** Given an array of primary asset ids, it loads all of the bundles referenced by properties of these assets specified in the LoadBundles array. */
	/** 给定一组主要资产标识符，它会加载由这些资产的属性中所指定的“加载包”数组所引用的所有包。*/
	void AsyncPreloadPrimaryAssetsAndBundles(const TArray<FPrimaryAssetId>& AssetIds, const TArray<FName>& LoadBundles,
	                                         TFunction<void()>&& Callback)
	{
		AsyncPreloadPrimaryAssetsAndBundles(AssetIds, LoadBundles, FSimpleDelegate::CreateLambda(MoveTemp(Callback)));
	}

```
``` cpp
	/** Given an array of primary asset ids, it loads all of the bundles referenced by properties of these assets specified in the LoadBundles array. */
	/** 给定一组主要资产标识符，它会加载由这些资产的属性中所指定的“加载包”数组所引用的所有包。*/
	UE_API void AsyncPreloadPrimaryAssetsAndBundles(const TArray<FPrimaryAssetId>& AssetIds,
	                                                const TArray<FName>& LoadBundles,
	                                                const FSimpleDelegate& Callback = FSimpleDelegate());
```

### 异步调用条件执行事件
``` cpp
	/** Add a future condition that must be true before we move forward. */
	/** 添加一个在未来必须为真的条件，只有在该条件为真时我们才能继续进行。*/
	UE_API void AsyncCondition(TSharedRef<FAsyncCondition> Condition,
	                           const FSimpleDelegate& Callback = FSimpleDelegate());
```

### 异步调用执行事件

``` cpp
	/**
	 * Rather than load anything, this callback is just inserted into the callback sequence so that when async loading 
	 * completes this event will be called at the same point in the sequence.  Super useful if you don't want a step to be
	 * tied to a particular asset in case some of the assets are optional.
	 */
	/**
	* 该回调函数并非用于加载任何内容，而是直接被插入到回调序列中，以便在异步加载完成后，此事件会在序列中的同一位置被触发。
	* 如果不想让某个步骤与特定的资源绑定（以防某些资源是可选的），那么使用这个回调功能会非常有用。
	* 
	 */
	void AsyncEvent(TFunction<void()>&& Callback)
	{
		AsyncEvent(FSimpleDelegate::CreateLambda(MoveTemp(Callback)));
	}

```
``` cpp
	/**
	 * Rather than load anything, this callback is just inserted into the callback sequence so that when async loading
	 * completes this event will be called at the same point in the sequence.  Super useful if you don't want a step to be
	 * tied to a particular asset in case some of the assets are optional.
	 */
	/**
	* 该回调函数并非用于加载任何内容，而是直接被插入到回调序列中，以便在异步加载完成后，此事件会在序列中的同一位置被触发。
	* 如果不想让某个步骤与特定的资源绑定（以防某些资源是可选的），那么使用这个回调功能会非常有用。*/
	UE_API void AsyncEvent(const FSimpleDelegate& Callback);

```
### 加载控制
``` cpp
	/** Flushes any async loading requests. */
	/** 刷新所有异步加载请求。*/
	UE_API void StartAsyncLoading();

	/** Cancels any pending async loads. */
	/** 取消所有正在进行的异步加载操作。*/
	UE_API void CancelAsyncLoading();

	/** Is async loading current in progress? */
	/** 当前是否正在进行异步加载操作？*/
	UE_API bool IsAsyncLoadingInProgress() const;
```
#### 启动异步加载
``` cpp
void FAsyncMixin::StartAsyncLoading()
{
	// If we don't actually have any loading state because they've not queued anything to load,
	// just immediately start and finish the operation by calling the callbacks, no point in allocating
	// the memory just to de-allocate it.
	// 如果我们实际上并没有任何加载状态（因为他们尚未安排任何需要加载的内容），那么就直接调用回调函数来立即启动并完成操作，因为没必要先分配内存然后再释放它。
	if (IsLoadingInProgressOrPending())
	{
		GetLoadingState().Start();
	}
	else
	{
		OnStartedLoading();
		OnFinishedLoading();
	}
}
```
#### 取消异步加载
``` cpp
void FAsyncMixin::CancelAsyncLoading()
{
	// Don't create the loading state if we don't have anything pending.
	// 如果没有待处理的任务，就不要创建加载状态。
	if (HasLoadingState())
	{
		GetLoadingState().CancelAndDestroy();
	}
}
```
#### 是否正在加载
``` cpp
bool FAsyncMixin::IsAsyncLoadingInProgress() const
{
	// Don't create the loading state if we don't have anything pending.
	// 若当前没有待处理的任务，则不要创建加载状态。
	if (HasLoadingState())
	{
		return GetLoadingStateConst().IsLoadingInProgress();
	}

	return false;
}

```

### 转发到内部处理
``` cpp
void FAsyncMixin::AsyncLoad(FSoftObjectPath SoftObjectPath, const FSimpleDelegate& DelegateToCall)
{
	GetLoadingState().AsyncLoad(SoftObjectPath, DelegateToCall);
}

void FAsyncMixin::AsyncLoad(const TArray<FSoftObjectPath>& SoftObjectPaths, const FSimpleDelegate& DelegateToCall)
{
	GetLoadingState().AsyncLoad(SoftObjectPaths, DelegateToCall);
}

void FAsyncMixin::AsyncPreloadPrimaryAssetsAndBundles(const TArray<FPrimaryAssetId>& AssetIds, const TArray<FName>& LoadBundles, const FSimpleDelegate& DelegateToCall)
{
	GetLoadingState().AsyncPreloadPrimaryAssetsAndBundles(AssetIds, LoadBundles, DelegateToCall);
}

void FAsyncMixin::AsyncCondition(TSharedRef<FAsyncCondition> Condition, const FSimpleDelegate& Callback)
{
	GetLoadingState().AsyncCondition(Condition, Callback);
}

void FAsyncMixin::AsyncEvent(const FSimpleDelegate& Callback)
{
	GetLoadingState().AsyncEvent(Callback);
}
```
## 工具函数
``` cpp
	// 获取我们指向的加载状态
	UE_API const FLoadingState& GetLoadingStateConst() const;
	// 获取或者创建一个加载状态
	UE_API FLoadingState& GetLoadingState();

	// 是否由加载状态
	UE_API bool HasLoadingState() const;

	// 是否正出意加载中 只是询问 避免创建
	UE_API bool IsLoadingInProgressOrPending() const;
```
## FLoadingState
``` cpp
/**
	 * The FLoadingState is what actually is allocated for the FAsyncMixin in a big map so that the FAsyncMixin itself holds no
	 * no memory, and we dynamically create the FLoadingState only if needed, and destroy it when it's unneeded.
	 * “FLoadingState” 实际上是为 FAsyncMixin 在一个大型映射中所分配的内存区域，这样 FAsyncMixin 自身就不需要持有任何内存了。
	 * 我们仅在需要时动态创建“FLoadingState”，并在不再需要时将其销毁。
	 * 
	 */
	class FLoadingState : public TSharedFromThis<FLoadingState>
	{
	public:
		// 构造函数 无作用 传递参数
		FLoadingState(FAsyncMixin& InOwner);
		// 很重要 取消所有异步任务 取消可能之前我们自己使用时设置的回收内存 因为我们已经即将要毁灭了
		virtual ~FLoadingState();

		/** Starts the async sequence. */
		// 开始异步序列
		void Start();

		/** Cancels the async sequence. */
		// 取消异步序列
		void CancelAndDestroy();
		
		// 异步加载一个资产
		void AsyncLoad(FSoftObjectPath SoftObject, const FSimpleDelegate& DelegateToCall);
		// 异步加载一组资产
		void AsyncLoad(const TArray<FSoftObjectPath>& SoftObjectPaths, const FSimpleDelegate& DelegateToCall);
		// 异步加载一组主要资产
		void AsyncPreloadPrimaryAssetsAndBundles(const TArray<FPrimaryAssetId>& PrimaryAssetIds,
		                                         const TArray<FName>& LoadBundles,
		                                         const FSimpleDelegate& DelegateToCall);
		// 异步条件执行
		void AsyncCondition(TSharedRef<FAsyncCondition> Condition, const FSimpleDelegate& Callback);
		// 异步执行
		void AsyncEvent(const FSimpleDelegate& Callback);

		// 是否加载完成
		bool IsLoadingComplete() const { return !IsLoadingInProgress(); }
		// 是否在加载中 
		// 做了优化 避免循环 
		// 采用当前轮询的索引和任务数进行比较得出是否正在加载中
		bool IsLoadingInProgress() const;
		// 是否在加载中,或者等待中
		bool IsLoadingInProgressOrPending() const;
		// 是否等待销毁中
		bool IsPendingDestroy() const;

	private:
		// 只是取消混入流程 bDestroying是否实在析构时执行 即摧毁时
		void CancelOnly(bool bDestroying);
		// 取消下一帧启动的定时器
		void CancelStartTimer();
		
		// 尝试按计划启动
		void TryScheduleStart();
		// 尝试完成异步加载流程
		void TryCompleteAsyncLoading();
		void CompleteAsyncLoading();

	private:
		// 请求摧毁内存
		void RequestDestroyThisMemory();
		// 取消摧毁内存 bDestroying是否实在析构时执行 即摧毁时
		void CancelDestroyThisMemory(bool bDestroying);

		/** Who owns the loading state?  We need this to call back into the owning mix-in object. */
		/** 谁拥有加载状态呢？我们需要这个信息以便能够回调到拥有该加载状态的混入对象中。*/
		FAsyncMixin& OwnerRef;

		/**
		 * Did we need to pre-load bundles?  If we didn't pre-load bundles (which require you keep the streaming handle 
		 * around or they will be destroyed), then we can safely destroy the FLoadingState when everything is done loading.
		 */
		/**
		* 我们是否需要预先加载包呢？如果我们不预先加载包（因为这要求您保留流处理句柄，否则这些包将会被销毁），那么在所有加载工作完成后，我们可以放心地销毁 FLoadingState 对象。*/
		bool bPreloadedBundles = false;
		// 是否已经启动了 标志位 需要避免多次启动 以及确认执行完成后回调时 是在正确的已经启动的情况下
		bool bHasStarted = false;

		// 当前轮询任务玩家完成进度的索引
		int32 CurrentAsyncStep = 0;
		// 所有的异步任务
		TArray<TUniquePtr<FAsyncStep>> AsyncSteps;
		// 等待回收的异步任务
		TArray<TUniquePtr<FAsyncStep>> AsyncStepsPendingDestruction;
		// 确保下一帧能启动的代理句柄
		FTSTicker::FDelegateHandle StartTimerDelegate;
		// 用于下一帧进行内存回收的句柄
		FTSTicker::FDelegateHandle DestroyMemoryDelegate;
	}	
```
## FAsyncStep
``` cpp
		class FAsyncStep
		{
		public:
			// 异步任务 调用执行
			FAsyncStep(const FSimpleDelegate& InUserCallback);
			// 异步任务 加载资产 调用执行
			FAsyncStep(const FSimpleDelegate& InUserCallback, const TSharedPtr<FStreamableHandle>& InStreamingHandle);
			// 异步任务 有条件的 调用执行
			FAsyncStep(const FSimpleDelegate& InUserCallback, const TSharedPtr<FAsyncCondition>& InCondition);
			// 析构函数 无
			~FAsyncStep();

			// 执行用户回调
			void ExecuteUserCallback();

			// 是否在加载中
			bool IsLoadingInProgress() const
			{
				return !IsComplete();
			}
			// 是否已经完成 
			// 加载句柄可用则使用加载句柄的结果
			// 条件可用个就使用条件的执行结果
			bool IsComplete() const;
			
			// 取消本任务 并重置完成后回调
			void Cancel();

			// 绑定本任务执行完成后的回调,注意应当避免已经完成之后再绑定的情况
			bool BindCompleteDelegate(const FSimpleDelegate& NewDelegate);
			// 是否绑定了完成后回调
			bool IsCompleteDelegateBound() const;

		private:
			// 绑定的用户回调
			FSimpleDelegate UserCallback;
			// 是否绑定了完成回调 注意是完成回调,不是用户回调!!!
			bool bIsCompletionDelegateBound = false;

			// Possible Async 'thing'
			// 用于处理可能的异步句柄
			// 流式资产记载
			TSharedPtr<FStreamableHandle> StreamingHandle;
			// 条件执行
			TSharedPtr<FAsyncCondition> Condition;
		};

```
## FAsyncScope
你不想继承的时候 用这个就行
``` cpp
/**
 * Sometimes a mix-in just doesn't make sense.  Perhaps the object has to manage many different jobs
 * that each have their own async dependency chain/scope.  For those situations you can use the FAsyncScope.
 * 
 * This class is a standalone Async dependency handler so that you can fire off several load jobs and always handle them
 * in the proper order, just like with combining FAsyncMixin with your class.
 * 
 * 有时，混合使用方式并不合适。或许该对象需要承担多种不同的任务，
 * 而且每项任务都有各自的异步依赖链/作用域。对于这类情况，您可以使用 FAsyncScope 。*
 * 此类是一个独立的异步依赖处理程序，这样您就可以启动多个加载任务，并且能够始终按照正确的顺序处理它们，就像将 FAsyncMixin 与您的类结合使用时那样。
 * 
 */
class FAsyncScope : public FAsyncMixin
{
public:
	using FAsyncMixin::AsyncLoad;

	using FAsyncMixin::AsyncPreloadPrimaryAssetsAndBundles;

	using FAsyncMixin::AsyncCondition;

	using FAsyncMixin::AsyncEvent;

	using FAsyncMixin::CancelAsyncLoading;

	using FAsyncMixin::StartAsyncLoading;

	using FAsyncMixin::IsAsyncLoadingInProgress;
};

//------------------------------------------------------------------------------
//------------------------------------------------------------------------------

```

## FAsyncCondition
``` cpp
//------------------------------------------------------------------------------
//------------------------------------------------------------------------------
// 函数返回的枚举,根据这个枚举决定是否再次执行
enum class EAsyncConditionResult : uint8
{
	// 再次尝试
	TryAgain,
	// 已完成
	Complete
};

DECLARE_DELEGATE_RetVal(EAsyncConditionResult, FAsyncConditionDelegate);

/**
 * The async condition allows you to have custom reasons to hault the async loading until some condition is met.
 * 异步条件使您能够设定自定义的停止条件，从而在满足某些条件时暂停异步加载过程。
 */
class FAsyncCondition : public TSharedFromThis<FAsyncCondition>
{
public:
	FAsyncCondition(const FAsyncConditionDelegate& Condition);
	FAsyncCondition(TFunction<EAsyncConditionResult()>&& Condition);
	virtual ~FAsyncCondition();

protected:
	// 是否已经完成 通过多次调用UserCondition的函数获得返回值
	bool IsComplete() const;
	// 绑定完成后回调,会设置一个定时器去轮询
	bool BindCompleteDelegate(const FSimpleDelegate& NewDelegate);

private:
	
	// 定时器轮询用户条件的函数
	bool TryToContinue(float DeltaTime);

	FTSTicker::FDelegateHandle RepeatHandle;
	// 用户定义的条件
	FAsyncConditionDelegate UserCondition;
	// 完成后回调,在自己的定时器轮询到结果后调用
	FSimpleDelegate CompletionDelegate;

	friend FAsyncMixin;
};

```

## 调用流程

### 异步加载任务
#### 函数入口
``` cpp

	/** Async load an array of FSoftObjectPath, call the Callback when complete. */
	/** 异步加载一个包含 FSoftObjectPath 的数组，并在完成时调用回调函数。*/
	UE_API void AsyncLoad(const TArray<FSoftObjectPath>& SoftObjectPaths,
	                      const FSimpleDelegate& Callback = FSimpleDelegate());
```

#### 获取加载状态
``` cpp
TMap<FAsyncMixin*, TSharedRef<FAsyncMixin::FLoadingState>> FAsyncMixin::Loading;

FAsyncMixin::FLoadingState& FAsyncMixin::GetLoadingState()
{
	check(IsInGameThread());

	if (TSharedRef<FLoadingState>* LoadingState = Loading.Find(this))
	{
		return (*LoadingState).Get();
	}

	return Loading.Add(this, MakeShared<FLoadingState>(*this)).Get();
}
```
#### 转发
``` cpp
void FAsyncMixin::AsyncLoad(const TArray<FSoftObjectPath>& SoftObjectPaths, const FSimpleDelegate& DelegateToCall)
{
	GetLoadingState().AsyncLoad(SoftObjectPaths, DelegateToCall);
}

```

#### 创建异步加载任务
``` cpp
void FAsyncMixin::FLoadingState::AsyncLoad(const TArray<FSoftObjectPath>& SoftObjectPaths, const FSimpleDelegate& DelegateToCall)
{
	{
		const FString& Paths = FString::JoinBy(SoftObjectPaths, TEXT(", "), [](const FSoftObjectPath& SoftObjectPath) { return FString::Printf(TEXT("'%s'"), *SoftObjectPath.ToString()); });
		UE_LOG(LogAsyncMixin, Verbose, TEXT("[0x%X] AsyncLoad [%s]"), this, *Paths);
	}

	AsyncSteps.Add(
		MakeUnique<FAsyncStep>(
			DelegateToCall,
			UAssetManager::GetStreamableManager().RequestAsyncLoad(SoftObjectPaths, FStreamableDelegate(), FStreamableManager::AsyncLoadHighPriority, false, false, TEXT("AsyncMixin"))
			)
	);

	TryScheduleStart();
}

```
#### 确保下一帧能启动
``` cpp
void FAsyncMixin::FLoadingState::TryScheduleStart()
{
	CancelDestroyThisMemory(/*bDestroying*/false);

	// In the event the user forgets to start async loading, we'll begin doing it next frame.
	// 如果用户忘记启动异步加载功能，我们将会在下一帧开始进行加载操作。
	if (!StartTimerDelegate.IsValid())
	{
		StartTimerDelegate = FTSTicker::GetCoreTicker().AddTicker(FTickerDelegate::CreateLambda([this](float DeltaTime) {
			QUICK_SCOPE_CYCLE_COUNTER(STAT_FAsyncMixin_FLoadingState_TryScheduleStartDelegate);
			Start();
			return false;
		}));
	}
}

```
#### 取消之前安排的回收操作
``` cpp
void FAsyncMixin::FLoadingState::CancelDestroyThisMemory(bool bDestroying)
{
	// If we've schedule the memory to be deleted we need to abort that.
	// 如果我们已经安排了要删除的内存操作，那么就需要取消这一操作。
	if (IsPendingDestroy())
	{
		if (!bDestroying)
		{
			UE_LOG(LogAsyncMixin, Verbose, TEXT("[0x%X] Destroy LoadingState (Canceled)"), this);
		}

		FTSTicker::GetCoreTicker().RemoveTicker(DestroyMemoryDelegate);
		DestroyMemoryDelegate.Reset();
	}
}

```
#### 轮询是否加载完成

``` cpp
void FAsyncMixin::FLoadingState::Start()
{
	UE_LOG(LogAsyncMixin, Verbose, TEXT("[0x%X] Start (Current Progress %d/%d)"), this, CurrentAsyncStep + 1, AsyncSteps.Num());

	// Cancel any pending kickoff load requests.
	// 取消所有正在等待的启动加载请求。
	CancelStartTimer();

	bool bStartingStepFound = false;

	if (!bHasStarted)
	{
		bHasStarted = true;
		OwnerRef.OnStartedLoading();
	}
	
	TryCompleteAsyncLoading();
}
```
我们可以手动调用 或者等待下一帧自动启动 
以下是手动调用的位置
``` cpp
void FAsyncMixin::StartAsyncLoading()
{
	// If we don't actually have any loading state because they've not queued anything to load,
	// just immediately start and finish the operation by calling the callbacks, no point in allocating
	// the memory just to de-allocate it.
	// 如果我们实际上并没有任何加载状态（因为他们尚未安排任何需要加载的内容），那么就直接调用回调函数来立即启动并完成操作，因为没必要先分配内存然后再释放它。
	if (IsLoadingInProgressOrPending())
	{
		GetLoadingState().Start();
	}
	else
	{
		OnStartedLoading();
		OnFinishedLoading();
	}
}


```
取消启动的定时器,避免多次启动
``` cpp
void FAsyncMixin::FLoadingState::CancelStartTimer()
{
	if (StartTimerDelegate.IsValid())
	{
		FTSTicker::GetCoreTicker().RemoveTicker(StartTimerDelegate);
		StartTimerDelegate.Reset();
	}
}

```

#### 加载完成执行回调
``` cpp

void FAsyncMixin::FLoadingState::TryCompleteAsyncLoading()
{
	// If we haven't started when we get this callback it means we've already completed
	// and this is some other callback finishing on the same frame/stack that we need to avoid
	// doing anything with until the memory is finished being deleted.
	// 如果在收到此回调时我们尚未开始操作，那就意味着我们已经完成了相关操作。
	// 而这个回调是在同一帧/栈上执行的，因此我们需要避免在此期间进行任何操作，直到内存被完全删除完毕。
	if (!bHasStarted)
	{
		return;
	}

	UE_LOG(LogAsyncMixin, Verbose, TEXT("[0x%X] TryCompleteAsyncLoading - (Current Progress %d/%d)"), this, CurrentAsyncStep + 1, AsyncSteps.Num());

	while (CurrentAsyncStep < AsyncSteps.Num())
	{
		FAsyncStep* Step = AsyncSteps[CurrentAsyncStep].Get();
		if (Step->IsLoadingInProgress())
		{
			if (!Step->IsCompleteDelegateBound())
			{
				UE_LOG(LogAsyncMixin, Verbose, TEXT("[0x%X] Step %d - Still Loading (Listening)"), this, CurrentAsyncStep + 1);
				const bool bBound = Step->BindCompleteDelegate(FSimpleDelegate::CreateSP(this, &FLoadingState::TryCompleteAsyncLoading));
				ensureMsgf(bBound, TEXT("This is not intended to return false.  We're checking if it's loaded above, this should definitely return true."));
			}
			else
			{
				UE_LOG(LogAsyncMixin, Verbose, TEXT("[0x%X] Step %d - Still Loading (Waiting)"), this, CurrentAsyncStep + 1);
			}

			break;
		}
		else
		{
			UE_LOG(LogAsyncMixin, Verbose, TEXT("[0x%X] Step %d - Completed (Calling User)"), this, CurrentAsyncStep + 1);

			// Always advance the CurrentAsyncStep, before calling the user callback, it's possible they might
			// add new work, and try and start again, so we need to be ready for the next bit.
			// 在调用用户回调函数之前，一定要先推进当前异步步骤。因为用户有可能会添加新的工作并尝试重新开始，所以我们需要做好准备迎接接下来的环节。
			CurrentAsyncStep++;

			Step->ExecuteUserCallback();
		}
	}
	
	// If we're done loading, and bHasStarted is still true (meaning this is the first time we're encountering a request to complete)
	// try and complete.  It's entirely possible that a user callback might append new work, which they immediately start, which
	// immediately tries to complete, which might create a case where we're now inside of TryCompleteAsyncLoading, which then
	// calls Start, which then calls TryCompleteAsyncLoading, so when we come back out of the stack, we need to avoid trying to
	// complete the async loading N+ times.
	// 如果加载已完成，而 bHasStarted 仍为真（这意味着这是我们首次遇到需要完成请求的情况）
	// 尝试完成操作。
	// 有可能用户回调会添加新的工作，并立即开始处理，紧接着又立即尝试完成，这可能会导致我们此时处于 TryCompleteAsyncLoading 函数内部，然后该函数会调用 Start 函数，接着再调用 TryCompleteAsyncLoading 函数，所以当我们从栈中退出时，需要避免重复尝试完成异步加载 N 次。
	if (IsLoadingComplete() && bHasStarted)
	{
		CompleteAsyncLoading();
	}
}
```

#### 绑定处理尝试完成函数的重入
``` cpp

bool FAsyncMixin::FLoadingState::FAsyncStep::BindCompleteDelegate(const FSimpleDelegate& NewDelegate)
{
	// 已经完成了 不需要绑定.
	// 你绑定的时候 就应该访问一下
	if (IsComplete())
	{
		// Too Late!
		return false;
	}

	if (StreamingHandle.IsValid())
	{
		StreamingHandle->BindCompleteDelegate(NewDelegate);
	}
	else if (Condition)
	{
		Condition->BindCompleteDelegate(NewDelegate);
	}

	// 完成了执行完成后的绑定
	bIsCompletionDelegateBound = true;

	return true;
}

```
##### 资产的异步加载回调
``` cpp
bool FStreamableHandle::BindCompleteDelegate(FStreamableDelegate NewDelegate)
{
	if (!IsLoadingInProgress())
	{
		// Too late!
		return false;
	}

	return BindCompleteDelegate(
		UE::StreamableManager::Private::WrapDelegate(MoveTemp(NewDelegate)));
}
```
##### 用户自己定义的条件
``` cpp
bool FAsyncCondition::BindCompleteDelegate(const FSimpleDelegate& NewDelegate)
{
	if (IsComplete())
	{
		// Already Complete
		return false;
	}

	CompletionDelegate = NewDelegate;

	if (!RepeatHandle.IsValid())
	{
		// 创建轮询用户条件的定时器
		RepeatHandle = FTSTicker::GetCoreTicker().AddTicker(FTickerDelegate::CreateSP(this, &FAsyncCondition::TryToContinue), 0.16);
	}

	return true;
}
```
``` cpp
bool FAsyncCondition::TryToContinue(float)
{
	QUICK_SCOPE_CYCLE_COUNTER(STAT_FAsyncCondition_TryToContinue);

	UE_LOG(LogAsyncMixin, Verbose, TEXT("[0x%X] AsyncCondition::TryToContinue"), this);

	if (UserCondition.IsBound())
	{
		const EAsyncConditionResult Result = UserCondition.Execute();

		switch (Result)
		{
		case EAsyncConditionResult::TryAgain:
			return true;
		case EAsyncConditionResult::Complete:
			RepeatHandle.Reset();
			UserCondition.Unbind();

			CompletionDelegate.ExecuteIfBound();
			CompletionDelegate.Unbind();
			break;
		}
	}

	return false;
}
```
#### 该任务执行完成回调
``` cpp
void FAsyncMixin::FLoadingState::FAsyncStep::ExecuteUserCallback()
{
	UserCallback.ExecuteIfBound();
	UserCallback.Unbind();
}

```

#### 回收资源
``` cpp
void FAsyncMixin::FLoadingState::CompleteAsyncLoading()
{
	UE_LOG(LogAsyncMixin, Verbose, TEXT("[0x%X] CompleteAsyncLoading"), this);

	// Mark that we've completed loading.
	// 表示我们已完成加载操作。
	if (bHasStarted)
	{
		bHasStarted = false;
		OwnerRef.OnFinishedLoading();
	}

	// It's unlikely but possible they started loading more stuff in the OnFinishedLoading callback,
	// so double check that we're still actually done.
	//
	// NOTE: We don't delete ourselves from memory in use.  Doing things like
	// pre-loading a bundle requires keeping the streaming handle alive.  So we're keeping
	// things alive.
	// 
	// We won't destroy the memory but we need to cleanup anything that may be hanging on to
	// captured scope, like completion handlers.
	// 这种情况不太可能发生，但也不排除他们是在“加载完成回调函数”中加载了更多内容的可能性，
	// 所以要再次确认我们实际上已经完成了加载操作。//
	// 注意：在使用过程中，我们不会从内存中删除自身。例如，进行诸如预加载包之类的操作就需要保持流处理句柄处于活动状态。因此，我们会确保相关元素保持活跃状态。//
	// 我们不会破坏原有的内存，但我们需要清理掉任何可能仍残留着的已捕获的范围信息，比如完成处理程序等。
	if (IsLoadingComplete())
	{
		if (!bPreloadedBundles && !IsLoadingInProgressOrPending())
		{
			// If we're all done loading or pending loading, we should clean up the memory we're using.
			// go ahead and remove this loading state the owner mix-in allocated.
			// 如果我们已完成加载或正处于加载状态，则应当清理我们所使用的内存。
			// 现在可以删除由拥有者混入机制所分配的此加载状态了。
			RequestDestroyThisMemory();
			return;
		}
	}
}

```
#### 内存回收
``` cpp
void FAsyncMixin::FLoadingState::RequestDestroyThisMemory()
{
	// If we're already pending to destroy this memory, just ignore.
	// 如果我们已经确定要销毁这段内存，那就直接忽略掉吧。
	if (!IsPendingDestroy())
	{
		UE_LOG(LogAsyncMixin, Verbose, TEXT("[0x%X] Destroy LoadingState (Requested)"), this);
		// 去下一帧执行即可
		DestroyMemoryDelegate = FTSTicker::GetCoreTicker().AddTicker(FTickerDelegate::CreateLambda([this](float DeltaTime) {
			// Remove any memory we were using.
			//  清除我们所占用的任何内存。
			FAsyncMixin::Loading.Remove(&OwnerRef);
			// 不需要再次执行了
			return false;
		}));
	}
}

```
### 异步加载主要资产
``` cpp

void FAsyncMixin::AsyncPreloadPrimaryAssetsAndBundles(const TArray<FPrimaryAssetId>& AssetIds, const TArray<FName>& LoadBundles, const FSimpleDelegate& DelegateToCall)
{
	GetLoadingState().AsyncPreloadPrimaryAssetsAndBundles(AssetIds, LoadBundles, DelegateToCall);
}

```
``` cpp
void FAsyncMixin::FLoadingState::AsyncPreloadPrimaryAssetsAndBundles(const TArray<FPrimaryAssetId>& AssetIds, const TArray<FName>& LoadBundles, const FSimpleDelegate& DelegateToCall)
{
	{		
		const FString& Assets = FString::JoinBy(AssetIds, TEXT(", "), [](const FPrimaryAssetId& AssetId) { return AssetId.ToString(); });
		const FString& Bundles = FString::JoinBy(LoadBundles, TEXT(", "), [](const FName& LoadBundle) { return LoadBundle.ToString(); });
		UE_LOG(LogAsyncMixin, Verbose, TEXT("[0x%X]  AsyncPreload Assets [%s], Bundles[%s]"), this, *Assets, *Bundles);
	}

	TSharedPtr<FStreamableHandle> StreamingHandle;

	if (AssetIds.Num() > 0)
	{
		// 这个标志位 如果开启了,正常使用时就不会内存回收
		bPreloadedBundles = true;

		const bool bLoadRecursive = true;
		StreamingHandle = UAssetManager::Get().PreloadPrimaryAssets(AssetIds, LoadBundles, bLoadRecursive);
	}

	AsyncSteps.Add(MakeUnique<FAsyncStep>(DelegateToCall, StreamingHandle));

	TryScheduleStart();
}

```
### 异步条件执行
``` cpp
void FAsyncMixin::AsyncCondition(TSharedRef<FAsyncCondition> Condition, const FSimpleDelegate& Callback)
{
	GetLoadingState().AsyncCondition(Condition, Callback);
}

```
``` cpp
void FAsyncMixin::FLoadingState::AsyncCondition(TSharedRef<FAsyncCondition> Condition, const FSimpleDelegate& DelegateToCall)
{
	UE_LOG(LogAsyncMixin, Verbose, TEXT("[0x%X] AsyncCondition '0x%X'"), this, &Condition.Get());

	AsyncSteps.Add(MakeUnique<FAsyncStep>(DelegateToCall, Condition));

	TryScheduleStart();
}
```
### 异步执行
``` cpp
void FAsyncMixin::AsyncEvent(const FSimpleDelegate& Callback)
{
	GetLoadingState().AsyncEvent(Callback);
}
```
``` cpp
void FAsyncMixin::FLoadingState::AsyncEvent(const FSimpleDelegate& DelegateToCall)
{
	UE_LOG(LogAsyncMixin, Verbose, TEXT("[0x%X] AsyncEvent"), this);

	AsyncSteps.Add(MakeUnique<FAsyncStep>(DelegateToCall));

	TryScheduleStart();
}

```
``` cpp
void FAsyncMixin::FLoadingState::TryCompleteAsyncLoading()
{
	// ....
	// 轮询任务列表
	while (CurrentAsyncStep < AsyncSteps.Num())
	{
		FAsyncStep* Step = AsyncSteps[CurrentAsyncStep].Get();
		// 异步任务是否正在加载中
		// 注意异步Event 默认就是true 状态 所以就执行了!
		if (Step->IsLoadingInProgress())
		{
			// ...
		}
		else
		{
			// 这个任务执行完了
			UE_LOG(LogAsyncMixin, Verbose, TEXT("[0x%X] Step %d - Completed (Calling User)"), this, CurrentAsyncStep + 1);

			// Always advance the CurrentAsyncStep, before calling the user callback, it's possible they might
			// add new work, and try and start again, so we need to be ready for the next bit.
			// 在调用用户回调函数之前，一定要先推进当前异步步骤。因为用户有可能会添加新的工作并尝试重新开始，所以我们需要做好准备迎接接下来的环节。
			CurrentAsyncStep++;

			Step->ExecuteUserCallback();
		}

	}|
	// ....

}
```
## 总结
该插件就这两个文件,头文件500行,600行...
注意它这种写法,内部定义的结构体,只在内部使用.
这块的内容,跟我们UEC++给大家讲解多线程自定义写了一个大量任务任务框架是类似的.
同样的内容还有ControlFlow.
大家在学习的时候时候,要由易到难,没有必要强行要求自己手搓这样一套框架.而应当是在学会使用大量框架之后,再根据具体业务场景进行抽取使用即可!!!