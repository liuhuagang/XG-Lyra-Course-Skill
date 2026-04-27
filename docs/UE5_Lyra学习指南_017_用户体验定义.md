# UE5_Lyra学习指南_017_用户体验定义

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_017\_用户体验定义](#ue5_lyra学习指南_017_用户体验定义)
	- [概述](#概述)
	- [会话](#会话)
	- [回放](#回放)
	- [资产管理器配置](#资产管理器配置)
	- [代码](#代码)
	- [总结](#总结)



## 概述
本节我们将定义用户面临的体验.其实这个地方应该翻译的信达雅一些.  
其实呢,就是Exprience和UserFacingExperience的区别.  
UserFacingExperience是玩家玩游戏所关心的关卡内容,比如关卡名称,图标,主题等.  
Experience是我们开发时用来组织管理游戏玩法设计的结构.
前者是对后者进一步的封装使用.
## 会话
此处流程需要结合前端UMG来配套使用.后续章节讲解.

```cpp
public:
	/** Create a request object that is used to actually start a session with these settings */
	/** 创建一个请求对象，该对象用于根据这些设置实际启动会话 */
	UFUNCTION(BlueprintCallable, BlueprintPure=false, meta = (WorldContext = "WorldContextObject"))
	UCommonSession_HostSessionRequest* CreateHostingRequest(const UObject* WorldContextObject) const;
```
这个方法很重要,这会创建一个会话请求,后续会在GameMode中初始化DS服务器中用到,以及前端界面主持会话时用到.

## 回放

关于回放的内容也不在这里讲解

```cpp
/** Subsystem to handle recording/loading replays */
/** 用于处理录制/加载回放操作的子系统 */
UCLASS(MinimalAPI)
class ULyraReplaySubsystem : public UGameInstanceSubsystem
{
	GENERATED_BODY()

public:
	UE_API ULyraReplaySubsystem();

	/** Returns true if this platform supports replays at all */
	/** 返回值为真表示此平台完全支持回放功能 */
	UFUNCTION(BlueprintCallable, Category = Replays, BlueprintPure = false)
	static UE_API bool DoesPlatformSupportReplays();

	/** Returns the trait tag for platform support, used in options */
	/** 返回用于平台支持的特性标签，用于在选项中使用 */
	static UE_API FGameplayTag GetPlatformSupportTraitTag();
};
```
## 资产管理器配置
记得在引擎得资产管理配置这种类型得扫描资产,要不然引擎不知道,就不会扫到它.
``` txt
(PrimaryAssetType="LyraExperienceDefinition",AssetBaseClass="/Script/LyraGame.LyraExperienceDefinition",bHasBlueprintClasses=True,bIsEditorOnly=False,Directories=((Path="/Game/System/Experiences"),(Path="/Game/XGTest")),SpecificAssets=("/Game/System/FrontEnd/B_LyraFrontEnd_Experience.B_LyraFrontEnd_Experience"),Rules=(Priority=-1,ChunkId=-1,bApplyRecursively=True,CookRule=AlwaysCook))
```
记得两种Experience都要添加,这样才能找到!!!
``` txt
(PrimaryAssetType="LyraUserFacingExperienceDefinition",AssetBaseClass="/Script/LyraGame.LyraUserFacingExperienceDefinition",bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=((Path="/Game/UI/Temp"),(Path="/Game/System/Playlists")),SpecificAssets=,Rules=(Priority=-1,ChunkId=-1,bApplyRecursively=True,CookRule=AlwaysCook))

```
## 代码
``` cpp

/** Description of settings used to display experiences in the UI and start a new session */
/** 描述用于在用户界面中展示体验以及启动新会话所使用的设置 */
UCLASS(BlueprintType)
class ULyraUserFacingExperienceDefinition : public UPrimaryDataAsset
{
	GENERATED_BODY()

public:
	/** The specific map to load */
	/** 要加载的具体地图 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience, meta=(AllowedTypes="Map"))
	FPrimaryAssetId MapID;

	/** The gameplay experience to load */
	/** 正在加载的游戏体验内容 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience, meta=(AllowedTypes="LyraExperienceDefinition"))
	FPrimaryAssetId ExperienceID;

	/** Extra arguments passed as URL options to the game */
	/** 作为游戏的 URL 选项传递的额外参数 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience)
	TMap<FString, FString> ExtraArgs;

	/** Primary title in the UI */
	/** 用户界面中的主要标题 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience)
	FText TileTitle;

	/** Secondary title */
	/** 二级标题 */

	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience)
	FText TileSubTitle;

	/** Full description */
	/** 完整描述 */

	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience)
	FText TileDescription;

	/** Icon used in the UI */
	/** 在用户界面中使用的图标 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience)
	TObjectPtr<UTexture2D> TileIcon;

	/** The loading screen widget to show when loading into (or back out of) a given experience */
	/** 用于在进入（或退出）特定体验时显示的加载界面组件 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=LoadingScreen)
	TSoftClassPtr<UUserWidget> LoadingScreenWidget;

	/** If true, this is a default experience that should be used for quick play and given priority in the UI */
	/** 如果为真，则表示这是一个默认体验模式，适用于快速游玩，并且在用户界面中应予以优先展示 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience)
	bool bIsDefaultExperience = false;

	/** If true, this will show up in the experiences list in the front-end */
	/** 如果为真，则此内容将在前端的“经验”列表中显示 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience)
	bool bShowInFrontEnd = true;

	/** If true, a replay will be recorded of the game */
	/** 如果为真，则会记录游戏的回放过程 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience)
	bool bRecordReplay = false;

	/** Max number of players for this session */
	/** 本次游戏可参与的最大玩家数量 */
	UPROPERTY(BlueprintReadWrite, EditAnywhere, Category=Experience)
	int32 MaxPlayerCount = 16;

public:
	/** Create a request object that is used to actually start a session with these settings */
	/** 创建一个请求对象，该对象用于根据这些设置实际启动会话 */
	UFUNCTION(BlueprintCallable, BlueprintPure=false, meta = (WorldContext = "WorldContextObject"))
	UCommonSession_HostSessionRequest* CreateHostingRequest(const UObject* WorldContextObject) const;
};

```


## 总结
用户体验定义就是玩家可以直接进行阅读的关卡名片.