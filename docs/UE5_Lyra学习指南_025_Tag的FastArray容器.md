# UE5_Lyra学习指南_025_Tag的Fast TArray Replication

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_025\_Tag的Fast TArray Replication](#ue5_lyra学习指南_025_tag的fast-tarray-replication)
	- [概述](#概述)
	- [Fast TArray Replication](#fast-tarray-replication)
	- [使用步骤](#使用步骤)
		- [步骤1 使你的结构体继承自FFastArraySerializerItem](#步骤1-使你的结构体继承自ffastarrayserializeritem)
		- [步骤2 创建另一个结构体包含上一个结构体](#步骤2-创建另一个结构体包含上一个结构体)
		- [步骤3 生命周期和使用标记](#步骤3-生命周期和使用标记)
	- [网络序列化的概述及其工作原理](#网络序列化的概述及其工作原理)
	- [代码](#代码)
		- [GameplayTagStack.h](#gameplaytagstackh)
		- [GameplayTagStack.cpp](#gameplaytagstackcpp)
	- [总结](#总结)



## 概述
FastArray的写法比较固定.
就是分别继承FFastArraySerializerItem,FFastArraySerializer.
标记需要序列化的数据.
重写同步删除增加前后的方法.
记得每次修改容器之后需要标记被修改的数据.
记得在拥有者的地方重写生命周期的方法
## Fast TArray Replication
``` txt
/**
 *	===================== Fast TArray Replication ===================== 
 *
 *	Fast TArray Replication is a custom implementation of NetDeltaSerialize that is suitable for TArrays of UStructs. It offers performance
 *	improvements for large data sets, it serializes removals from anywhere in the array optimally, and allows events to be called on clients
 *	for adds and removals. The downside is that you will need to have game code mark items in the array as dirty, and well as the *order* of the list
 *	is not guaranteed to be identical between client and server in all cases.
 *
 *	Using FTR is more complicated, but this is the code you need:
 *
 */
```


===================== 快速数组复制 =====================
“快速 TArray 传输”是 NetDeltaSerialize 的一种自定义实现方式，适用于 UStruct 类型的 TArray。它能为大型数据集带来性能提升，能够最优地对数组中的任意位置进行删除操作，并允许在客户端调用添加和删除事件。但其缺点是，您需要在游戏代码中标记数组中的元素为“脏”，并且列表的顺序在所有情况下在客户端和服务器之间都不一定能保持一致。使用 FTR 则较为复杂，但你需要的代码是这样的：

## 使用步骤

### 步骤1 使你的结构体继承自FFastArraySerializerItem
``` cpp
/** Step 1: Make your struct inherit from FFastArraySerializerItem */
/** 步骤 1：让您的结构体继承自 FFastArraySerializerItem 类 */
USTRUCT()
struct FExampleItemEntry : public FFastArraySerializerItem
{
	GENERATED_USTRUCT_BODY()

	// Your data:
	// 你的数据
	UPROPERTY()
	int32		ExampleIntProperty;	

	UPROPERTY()
	float		ExampleFloatProperty;


	/** 
	 * Optional functions you can implement for client side notification of changes to items; 
	 * Parameter type can match the type passed as the 2nd template parameter in associated call to FastArrayDeltaSerialize
	 * 
	 * NOTE: It is not safe to modify the contents of the array serializer within these functions, nor to rely on the contents of the array 
	 * being entirely up-to-date as these functions are called on items individually as they are updated, and so may be called in the middle of a mass update.
	 */
	/**
	 * 您可以为客户端实现一些可选的功能，用于通知项目的变化；
	 * 参数类型应与在与 FastArrayDeltaSerialize 相关联的调用中作为第二个模板参数传递的类型相匹配。*
	 * 注意：在这些函数中修改数组序列化的内容是不安全的，而且也不应依赖于数组内容的完全更新状态，因为这些函数是在对单个项进行更新时调用的，所以可能会在大规模更新过程中被调用。
	 */
	void PreReplicatedRemove(const struct FExampleArray& InArraySerializer);
	void PostReplicatedAdd(const struct FExampleArray& InArraySerializer);
	void PostReplicatedChange(const struct FExampleArray& InArraySerializer);

	// Optional: debug string used with LogNetFastTArray logging
	// （可选）与 LogNetFastTArray 日志记录一起使用的调试字符串
	FString GetDebugString();

};

```
### 步骤2 创建另一个结构体包含上一个结构体

``` cpp
/** Step 2: You MUST wrap your TArray in another struct that inherits from FFastArraySerializer */
/** 第 2 步：您必须将您的 TArray 包装在一个继承自 FFastArraySerializer 的结构体中 */
USTRUCT()
struct FExampleArray: public FFastArraySerializer
{
	GENERATED_USTRUCT_BODY()

	UPROPERTY()
	TArray<FExampleItemEntry>	Items;	
	/** Step 3: You MUST have a TArray named Items of the struct you made in step 1. */
/** 第 3 步：您必须有一个名为“Items”的数组，其元素类型与您在第 1 步中定义的结构体相同。*/

	/** Step 4: Copy this, replace example with your names */
	/** 第 4 步：复制此代码，将“示例”部分替换为您的姓名 */
	bool NetDeltaSerialize(FNetDeltaSerializeInfo & DeltaParms)
	{
	   return FFastArraySerializer::FastArrayDeltaSerialize<FExampleItemEntry, FExampleArray>( Items, DeltaParms, *this );
	}
};


```
``` cpp
/** Step 5: Copy and paste this struct trait, replacing FExampleArray with your Step 2 struct. */
/** 第 5 步：复制并粘贴此结构体特性，将“FExampleArray”替换为您在第 2 步中定义的结构体名称。*/
template<>
struct TStructOpsTypeTraits< FExampleArray > : public TStructOpsTypeTraitsBase2< FExampleArray >
{
       enum 
       {
			WithNetDeltaSerializer = true,
       };
};



```

### 步骤3 生命周期和使用标记
``` txt
/** Step 6 and beyond: 
 *		-Declare a UPROPERTY of your FExampleArray (step 2) type.
 *		-You MUST call MarkItemDirty on the FExampleArray when you change an item in the array. You pass in a reference to the item you dirtied. 
 *			See FFastArraySerializer::MarkItemDirty.
 *		-You MUST call MarkArrayDirty on the FExampleArray if you remove something from the array.
 *		-In your classes GetLifetimeReplicatedProps, use DOREPLIFETIME(YourClass, YourArrayStructPropertyName);
 *
 *		You can provide the following functions in your structure (step 1) to get notifies before add/deletes/removes:
 *			-void PreReplicatedRemove(const FFastArraySerializer& Serializer)
 *			-void PostReplicatedAdd(const FFastArraySerializer& Serializer)
 *			-void PostReplicatedChange(const FFastArraySerializer& Serializer)
 *			-void PostReplicatedReceive(const FFastArraySerializer::FPostReplicatedReceiveParameters& Parameters)
 *
 *		Thats it!
 */ 
```

第 6 步及后续步骤：
		- 宣告一个具有“UPROPERTY”属性的您的 FExampleArray 类型（步骤 2）。
		- 当您更改数组中的元素时，必须对 FExampleArray 调用“MarkItemDirty”方法。您需要传递一个指向已修改元素的引用。
			请参阅 FFastArraySerializer:：MarkItemDirty。
		- 如果从数组中删除元素，则必须对 FExampleArray 调用“MarkArrayDirty”方法。
		- 在您的类的“GetLifetimeReplicatedProps”中，使用 DOREPLIFETIME(YourClass， YourArrayStructPropertyName)；*
在您的结构中（步骤 1）可以实现以下功能，以便在进行添加、删除或移除操作之前接收通知：
- void PreReplicatedRemove(const FFastArraySerializer& Serializer)
- void PostReplicatedAdd(const FFastArraySerializer& Serializer)
- void PostReplicatedChange(const FFastArraySerializer& Serializer)
- void PostReplicatedReceive(const FFastArraySerializer:：FPostReplicatedReceiveParameters& Parameters)
就这样！

## 网络序列化的概述及其工作原理
这部分内容可以不用看.搜知乎大佬们关于网络同步的讲解即可.
``` txt

/**
 *	
 *	===================== An Overview of Net Serialization and how this all works =====================
 *
 *	===================== 网络序列化概述及其工作原理详解 =====================
 *
 *		Everything originates in UNetDriver::ServerReplicateActors.
 *		一切皆始于 UNetDriver::ServerReplicateActors 这个函数。

 *		Actors are chosen to replicate, create actor channels, and UActorChannel::ReplicateActor is called.
 *		选定演员后，会创建演员通道，并调用 UActorChannel::ReplicateActor 方法。

 *		ReplicateActor is ultimately responsible for deciding what properties have changed, and constructing an FOutBunch to send to clients.
 *		ReplicateActor 最终负责确定哪些属性发生了变化，并构建一个 FOutBunch 以发送给客户端。
 *
 *	The UActorChannel has 2 ways to decide what properties need to be sent.
 *	UActorChannel 有两种方式来决定需要发送哪些属性。
 *	
 *		The traditional way, which is a flat TArray<uint8> buffer: UActorChannel::Recent. This represents a flat block of the actor properties.
 *		传统的实现方式是使用一个平坦的 `TArray<uint8>` 缓冲区：UActorChannel::Recent。这代表了该角色属性的一个平坦块。
 *		
 *		This block literally can be cast to an AActor* and property values can be looked up if you know the FProperty offset.
 *		这个块实际上可以转换为一个 AActor* 类型的变量，并且如果您知道 FProperty 的偏移量，就可以查找其属性值。
 *		
 *		The Recent buffer represents the values that the client using this actor channel has. We use recent to compare to current, and decide what to send.
 *		“近期缓冲区”代表了使用此代理通道的客户端所拥有的值。我们通过“近期”与“当前”进行比较，并据此决定要发送的内容。
 *
 *		This works great for 'atomic' properties; ints, floats, object*, etc.
 *		对于“原子”属性（如整数、浮点数、对象指针等）来说，这种方法效果极佳。
 *		
 *		It does not work for 'dynamic' properties such as TArrays, which store values Num/Max but also a pointer to their array data,
 *		但对于“动态”属性（如 TArrays）来说，这种方法并不适用。这类属性不仅存储数值（如 Num/Max），还存储指向其数组数据的指针。
 *		
 *		The array data has no where to fit in the flat ::Recent buffer. (Dynamic is probably a bad name for these properties)
 *		该数组数据无法放入“flat::Recent”缓冲区中。（“动态”这个名称或许不太适合这些属性）
 *
 *		To get around this, UActorChannel also has a TMap for 'dynamic' state. UActorChannel::RecentDynamicState. This map allows us to look up a 'base state' for a property given a property's RepIndex.
 *		为了解决这个问题，UActorChannel 还有一个用于“动态”状态的 TMap。即 UActorChannel:：RecentDynamicState。这个映射允许我们根据属性的 RepIndex 查找该属性的“基础状态”。
 *		
 *		
 *
 *	NetSerialize & NetDeltaSerialize
 *		Properties that fit into the flat Recent buffer can be serialized entirely with NetSerialize. NetSerialize just reads or writes to an FArchive.
 *		能够放入“近期缓冲区”的属性可以完全通过“NetSerialize”进行序列化处理。NetSerialize 只需对“FArchive”进行读取或写入操作即可。
 *		
 *		Since the replication can just look at the Recent[] buffer and do a direct comparison, it can tell what properties are dirty. NetSerialize just	reads or writes.
 *		由于复制操作只需查看“Recent[]”缓冲区并进行直接比较，就能确定哪些属性是需要更新的。而 NetSerialize 则只是进行读取或写入操作。
 *
 *		Dynamic properties can only be serialized with NetDeltaSerialize. NetDeltaSerialize is serialization from a given base state, and produces	both a 'delta' state (which gets sent to the client) and a 'full' state (which is saved to be used as the base state in future delta serializes).
 *		动态属性只能通过 NetDeltaSerialize 进行序列化。NetDeltaSerialize 是基于给定的基础状态进行序列化的操作，它会生成一个“差异”状态（该状态会发送给客户端）以及一个“完整”状态（该状态会被保存下来，以便在未来的差异序列化中用作基础状态）。

 *		NetDeltaSerialize essentially does the diffing as well as the serialization. It must do the diffing so it can know what parts of the property it must send.
 *		NetDeltaSerialize 主要负责进行差异计算以及数据序列化工作。它必须先完成差异计算，这样才能明确知道需要发送哪些属性的值。
 *	
 *	Base States and dynamic properties replication.
 *  基本状态与动态属性的复制。
 *		As far as the replication system / UActorChannel is concerned, a base state can be anything. The base state only deals with INetDeltaBaseState*.
 *		关于复制系统/ UActorChannel 而言，基础状态可以是任何内容。基础状态仅涉及 INetDeltaBaseState 。*
 *
 *		UActorChannel::ReplicateActor will ultimately decide whether to call FProperty::NetSerializeItem or FProperty::NetDeltaSerializeItem.
 *		UActorChannel::ReplicateActor 最终会决定是调用 FProperty::NetSerializeItem 还是 FProperty:：NetDeltaSerializeItem 。
 *
 *		As mentioned above NetDeltaSerialize takes in an extra base state and produces a diff state and a full state. The full state produced is used as the base state for future delta serialization. NetDeltaSerialize uses the base state and the current values of the actor to determine what parts it needs to send.
 *		如上所述，NetDeltaSerialize 接收一个额外的基础状态，并生成一个差异状态和一个完整状态。所生成的完整状态将用作后续差异序列化的基础状态。NetDeltaSerialize 利用基础状态以及该活动的当前值来确定需要发送哪些部分。
 *		
 *		The INetDeltaBaseStates are created within the NetDeltaSerialize functions. The replication system / UActorChannel does not know about the details.
 *		这些 INetDeltaBaseStates 是在 NetDeltaSerialize 函数中创建的。复制系统/ UActorChannel 并不了解其具体细节。
 *
 *		Right now, there are 2 forms of delta serialization: Generic Replication and Fast Array Replication.
 *		目前，有两种形式的“德尔塔”序列化方式：通用复制和快速阵列复制。
 *
 *	
 *	Generic Delta Replication
 *  通用德尔塔复制
 *		Generic Delta Replication is implemented by FStructProperty::NetDeltaSerializeItem, FArrayProperty::NetDeltaSerializeItem, FProperty::NetDeltaSerializeItem.
 *		通用差分复制是通过 FStructProperty::NetDeltaSerializeItem、FArrayProperty::NetDeltaSerializeItem 和 FProperty::NetDeltaSerializeItem 这些函数来实现的。
 *		
 *		It works by first NetSerializing the current state of the object (the 'full' state) and using memcmp to compare it to previous base state. 
 *		其工作原理是：首先对对象的当前状态（即“完整”状态）进行网络序列化处理，然后使用 memcmp 函数将其与之前的基状态进行比较。
 *		
 *		FProperty is what actually implements the comparison, writing the current state to the diff state if it has changed, and always writing to the full state otherwise.
 *		FProperty 实际上负责执行比较操作，如果当前状态有所变化，则会将该状态写入差异状态中，否则则始终将状态写入完整状态中。
 *		
 *		The FStructProperty and FArrayProperty functions work by iterating their fields or array elements and calling the FProperty function, while also embedding meta data. 
 *		FStructProperty 和 FArrayProperty 这两个函数的工作原理是：依次遍历其字段或数组元素，并调用 FProperty 函数，同时还会嵌入元数据。
 *
 *		For example FArrayProperty basically writes: 
 *			"Array has X elements now" -> "Here is element Y" -> Output from FProperty::NetDeltaSerialize -> "Here is element Z" -> etc
 *		
 *		例如，FArrayProperty 基本上会这样操作：
 *		“数组当前包含 X 个元素” -> “这里是第 Y 个元素” -> FProperty::NetDeltaSerialize 函数的输出结果 -> “这里是第 Z 个元素”等等
 *
 *		Generic Data Replication is the 'default' way of handling FArrayProperty and FStructProperty serialization. This will work for any array or struct with any sub properties as long as those properties can NetSerialize.
 *		通用数据复制是处理 FArrayProperty 和 FStructProperty 序列化的“默认”方式。只要这些属性能够进行网络序列化，那么无论数组或结构体中包含多少子属性，此方法都能适用。
 *
 *	Custom Net Delta Serialiation
 *  自定义网络差值序列化
 *		Custom Net Delta Serialiation works by using the struct trait system. If a struct has the WithNetDeltaSerializer trait, then its native NetDeltaSerialize function will be called instead of going through the Generic Delta Replication code path in FStructProperty::NetDeltaSerializeItem.
 *		Custom Net Delta Serialization 的实现方式是通过使用 struct 特性系统来实现的。如果一个结构体具有 WithNetDeltaSerializer 特性，那么其内置的 NetDeltaSerialize 函数将会被调用，而不会通过 FStructProperty:：NetDeltaSerializeItem 中的通用 Delta 传输代码路径来进行处理。
 *		
 *
 *	Fast TArray Replication
 *  快速数组复制
 *		Fast TArray Replication is implemented through custom net delta serialization. Instead of a flat TArray buffer to represent states, it only is concerned with a TMap of IDs and ReplicationKeys. The IDs map to items in the array, which all have a ReplicationID field defined in FFastArraySerializerItem.
 *		快速的数组复制是通过自定义网络差异序列化来实现的。它不再使用单一的平坦数组缓冲区来表示状态，而是只关注一个由 ID 和复制键组成的 TMap。这些 ID 对应于数组中的各个元素，而所有这些元素在 FFastArraySerializerItem 中都有一个定义了复制 ID 的字段。
 *		
 *		FFastArraySerializerItem also has a ReplicationKey field. When items are marked dirty with MarkItemDirty, they are given a new ReplicationKey, and assigned	a new ReplicationID if they don't have one.
 *		FFastArraySerializerItem 类还具有一个“ReplicationKey”字段。当使用“MarkItemDirty”方法标记某些项为已更改状态时，会为这些项分配一个新的“ReplicationKey”值，并且如果这些项之前没有“ReplicationID”值的话，还会为其分配一个新的“ReplicationID”值。
 *
 *		FastArrayDeltaSerialize (defined below)
 *		快速数组差值序列化（在下面已定义）
 *		During server serialization (writing), we compare the old base state (e.g, the old ID<->Key map) with the current state of the array. If items are missing we write them out as deletes in the bunch. If they are new or changed, they are written out as changed along with their state, serialized via a NetSerialize call.
 *		在服务器序列化（即写入）过程中，我们会将旧的基本状态（例如，旧的 ID <-> 键映射）与数组的当前状态进行比较。如果存在缺失的项，我们将它们作为删除操作一起写入序列化组中。如果这些项是新的或已更改的，则会将其作为已更改的内容一起写入，并通过 NetSerialize 调用对其进行序列化处理。
 *
 *		For example, what actually is written may look like:
 *			"Array has X changed elements, Y deleted elements" -> "element A changed" -> Output from NetSerialize on rest of the struct item -> "Element B was deleted" -> etc
  *		例如，实际所写的内容可能如下所示：
 *			“数组中有 X 个元素发生变化，Y 个元素被删除” -> “元素 A 发生了变化” -> 对该结构体其余部分进行序列化的输出结果 -> “元素 B 被删除” -> 等等
 *
 *		Note that the ReplicationID is replicated and in sync between client and server. The indices are not.
 *		请注意，复制标识符会在客户端和服务器之间进行同步复制。而索引则不会进行同步复制。
 *
 *		During client serialization (reading), the client reads in the number of changed and number of deleted elements. It also builds a mapping of ReplicationID -> local index of the current array.
 *		在客户端序列化（读取）过程中，客户端会读取已更改元素的数量和已删除元素的数量。同时，它还会构建一个“复制标识符” -> 当前数组本地索引的映射关系。
 *		
 *		As it deserializes IDs, it looks up the element and then does what it needs to (create if necessary, serialize in the current state, or delete).
 * 		在对其进行反序列化处理时，它会查找相应的元素，然后执行所需的操作（如果需要则创建该元素，或者根据当前状态进行序列化，或者删除该元素）。
 *
 *		Delta Serialization for inner structs is now enabled by default. That means that when a ReplicationKey changes, we will compare the current state of the struct to the last sent state, tracking changelists and only sending properties that changed exactly like the standard replication path.
 *		对于内部结构体的“德尔塔序列化”功能现在已默认开启。这意味着，当“复制键”发生变化时，我们将将该结构体的当前状态与上次发送的状态进行比较，跟踪变更列表，并仅发送那些与标准复制路径完全相同的属性，这与标准的复制路径方式一致。
 *		
 *		If this causes issues with a specific FastArray type, it can be disabled by calling FFastArraySerializer::SetDeltaSerializationEnabled(false) in the constructor.
 *		如果这导致了特定的 FastArray 类型出现问题，可以通过在构造函数中调用 FFastArraySerializer::SetDeltaSerializationEnabled(false) 来将其禁用。
 *		
 *		The feature can be completely disabled by setting the "net.SupportFastArrayDelta" CVar to 0.
 *		可以通过将“net.SupportFastArrayDelta”这个 CVar 设置为 0 来完全禁用该功能。
 *
 *		ReplicationID and ReplicationKeys are set by the MarkItemDirty function on FFastArraySerializer. These are just int32s that are assigned in order as things change.
 *		“复制标识”和“复制键”是由 FFastArraySerializer 中的 MarkItemDirty 函数设定的。它们只是整数类型的数据，会根据情况的变化按顺序进行赋值。
 *		There is nothing special about them other than being unique.
 *		它们本身并没有什么特别之处，只是独具特色而已。
 */


```





## 代码

### GameplayTagStack.h

``` cpp

/**
 * Represents one stack of a gameplay tag (tag + count)
 * 表示一个游戏玩法标签的一组数据（标签 + 数量）
 */
USTRUCT(BlueprintType)
struct FGameplayTagStack : public FFastArraySerializerItem
{
	GENERATED_BODY()

	FGameplayTagStack()
	{}

	FGameplayTagStack(FGameplayTag InTag, int32 InStackCount)
		: Tag(InTag)
		, StackCount(InStackCount)
	{
	}

	// 输出调试字段
	FString GetDebugString() const;

private:
	// 友元
	friend FGameplayTagStackContainer;

	// 使用的Tag
	UPROPERTY()
	FGameplayTag Tag;

	// Tag的数量
	UPROPERTY()
	int32 StackCount = 0;
};

/** Container of gameplay tag stacks */
/** 游戏玩法标签堆栈的容器 */
USTRUCT(BlueprintType)
struct FGameplayTagStackContainer : public FFastArraySerializer
{
	GENERATED_BODY()

	FGameplayTagStackContainer()
	//	: Owner(nullptr)
	{
	}

public:
	// Adds a specified number of stacks to the tag (does nothing if StackCount is below 1)
	// 向标签中添加指定数量的堆栈（若堆栈数量少于 1，则不执行任何操作）
	void AddStack(FGameplayTag Tag, int32 StackCount);

	// Removes a specified number of stacks from the tag (does nothing if StackCount is below 1)
	// 从标签中移除指定数量的堆栈（若“堆栈数量”小于 1，则不执行任何操作）
	void RemoveStack(FGameplayTag Tag, int32 StackCount);

	// Returns the stack count of the specified tag (or 0 if the tag is not present)
	// 返回指定标签的栈数量（若该标签不存在则返回 0）
	int32 GetStackCount(FGameplayTag Tag) const
	{
		return TagToCountMap.FindRef(Tag);
	}

	// Returns true if there is at least one stack of the specified tag
	// 如果存在指定标签的至少一个栈，则返回 true
	bool ContainsTag(FGameplayTag Tag) const
	{
		return TagToCountMap.Contains(Tag);
	}

	//~FFastArraySerializer contract
	void PreReplicatedRemove(const TArrayView<int32> RemovedIndices, int32 FinalSize);
	void PostReplicatedAdd(const TArrayView<int32> AddedIndices, int32 FinalSize);
	void PostReplicatedChange(const TArrayView<int32> ChangedIndices, int32 FinalSize);
	//~End of FFastArraySerializer contract

	bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms)
	{
		return FFastArraySerializer::FastArrayDeltaSerialize<FGameplayTagStack, FGameplayTagStackContainer>(Stacks, DeltaParms, *this);
	}

private:
	// Replicated list of gameplay tag stacks
	// 游戏玩法标签堆叠的复制列表
	UPROPERTY()
	TArray<FGameplayTagStack> Stacks;
	
	// Accelerated list of tag stacks for queries
	// 查询用标签栈的加速列表, 这个东西不同步,所以需要在同步前后进行手动修改
	TMap<FGameplayTag, int32> TagToCountMap;
};

template<>
struct TStructOpsTypeTraits<FGameplayTagStackContainer> : public TStructOpsTypeTraitsBase2<FGameplayTagStackContainer>
{
	enum
	{
		// 开启Delta同步功能!
		WithNetDeltaSerializer = true,
	};
};


```


### GameplayTagStack.cpp

``` cpp
//////////////////////////////////////////////////////////////////////
// FGameplayTagStack

FString FGameplayTagStack::GetDebugString() const
{
	return FString::Printf(TEXT("%sx%d"), *Tag.ToString(), StackCount);
}

//////////////////////////////////////////////////////////////////////
// FGameplayTagStackContainer

void FGameplayTagStackContainer::AddStack(FGameplayTag Tag, int32 StackCount)
{
	if (!Tag.IsValid())
	{
		FFrame::KismetExecutionMessage(TEXT("An invalid tag was passed to AddStack"), ELogVerbosity::Warning);
		return;
	}

	if (StackCount > 0)
	{
		for (FGameplayTagStack& Stack : Stacks)
		{
			if (Stack.Tag == Tag)
			{
				const int32 NewCount = Stack.StackCount + StackCount;
				Stack.StackCount = NewCount;
				TagToCountMap[Tag] = NewCount;
				// 修改了元素 必须标记为脏!
				MarkItemDirty(Stack);
				return;
			}
		}

		FGameplayTagStack& NewStack = Stacks.Emplace_GetRef(Tag, StackCount);
		// 新增了元素 必须标记为脏
		MarkItemDirty(NewStack);
		// 方便快捷查询
		TagToCountMap.Add(Tag, StackCount);
	}
}

void FGameplayTagStackContainer::RemoveStack(FGameplayTag Tag, int32 StackCount)
{
	if (!Tag.IsValid())
	{
		FFrame::KismetExecutionMessage(TEXT("An invalid tag was passed to RemoveStack"), ELogVerbosity::Warning);
		return;
	}

	//@TODO: Should we error if you try to remove a stack that doesn't exist or has a smaller count?
	//@待办事项：如果尝试删除一个并不存在或数量更少的栈，我们是否应该报错？
	if (StackCount > 0)
	{
		for (auto It = Stacks.CreateIterator(); It; ++It)
		{
			FGameplayTagStack& Stack = *It;
			if (Stack.Tag == Tag)
			{
				if (Stack.StackCount <= StackCount)
				{
					It.RemoveCurrent();
					TagToCountMap.Remove(Tag);
					// 移除了元素 必须标记整个数组脏了!
					MarkArrayDirty();
				}
				else
				{
					const int32 NewCount = Stack.StackCount - StackCount;
					Stack.StackCount = NewCount;
					TagToCountMap[Tag] = NewCount;
					// 修改了元素 必须标记它脏了!
					MarkItemDirty(Stack);
				}
				return;
			}
		}
	}
}

void FGameplayTagStackContainer::PreReplicatedRemove(const TArrayView<int32> RemovedIndices, int32 FinalSize)
{
	// 受到同步删除前执行
	for (int32 Index : RemovedIndices)
	{
		const FGameplayTag Tag = Stacks[Index].Tag;
		// 移除Map中的快速查询
		TagToCountMap.Remove(Tag);
	}
}

void FGameplayTagStackContainer::PostReplicatedAdd(const TArrayView<int32> AddedIndices, int32 FinalSize)
{
	// 收到同步添加后执行
	for (int32 Index : AddedIndices)
	{
		const FGameplayTagStack& Stack = Stacks[Index];
		// 增加Map中的快速查询
		TagToCountMap.Add(Stack.Tag, Stack.StackCount);
	}
}

void FGameplayTagStackContainer::PostReplicatedChange(const TArrayView<int32> ChangedIndices, int32 FinalSize)
{
	// 受到同步添加改变后执行
	for (int32 Index : ChangedIndices)
	{
		const FGameplayTagStack& Stack = Stacks[Index];
		TagToCountMap[Stack.Tag] = Stack.StackCount;
	}
}



```
## 总结
这块的知识很重要!!!!
这是从引擎源码和工程源码扒出来的高级知识!
如果面试官要考核你在网络同步方面的功力,绝对会问这块.这块算是比较深入的内容了.特别使涉及到ActorChannel了!
目前掌握这种固定写法会使用就行.后面我们在Lyra的库存系统中也会用到它!
