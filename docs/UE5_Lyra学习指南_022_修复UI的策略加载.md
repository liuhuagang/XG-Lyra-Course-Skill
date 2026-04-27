# UE5_Lyra学习指南_022_修复UI的策略加载

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_022\_修复UI的策略加载](#ue5_lyra学习指南_022_修复ui的策略加载)
	- [概述](#概述)
	- [确保Experience资产已设置](#确保experience资产已设置)
	- [覆写UI管理子系统](#覆写ui管理子系统)
		- [修改配置](#修改配置)
		- [代码](#代码)
			- [UGameUIManagerSubsystem](#ugameuimanagersubsystem)
			- [ULyraUIManagerSubsystem](#ulyrauimanagersubsystem)
	- [配置加载界面](#配置加载界面)
	- [总结](#总结)



## 概述
目前我们的项目虽然可以编译成功,也可以启动编辑器.
但是一旦运行就会报错.这是正常的.因为有些配置我们还没有做
## 确保Experience资产已设置
请确保资产管理器中对于Experience的检索定义已确认.否则会出现如下崩溃:
``` txt
Log          LogRemoteSession          Started listening on port 2049
Error        LogLyraExperience         /Game/XGTest/UEDPIE_0_L_Simple.L_Simple:PersistentLevel.LyraWorldSettings.DefaultGameplayExperience is /Game/XGTest/BP_Experience_XG.BP_Experience_XG_C but that failed to resolve into an asset ID (you might need to add a path to the Asset Rules in your game feature plugin or project settings
Log          LogLyraExperience         Identified experience LyraExperienceDefinition:B_LyraDefaultExperience (Source: Default)
Warning      LogOutputDevice           Script Stack (0 frames) :
Error        LogWindows                appError called: Assertion failed: AssetClass [File:G:\UPS\GitLab\LyraGroup\LyraInternal\Lyra\Source\LyraGame\GameModes\LyraExperienceManagerComponent.cpp] [Line: 91] 
Log          LogWindows                Windows GetLastError: 操作成功完成。 (0)
```
## 覆写UI管理子系统
### 修改配置
DefaultGame.ini
```ini
[/Script/LyraGame.LyraUIManagerSubsystem]
; 默认的UI管理策略
DefaultUIPolicyClass=/Game/UI/B_LyraUIPolicy.B_LyraUIPolicy_C

```
如果没有配置的话,会在这里崩溃!因为空指针了!
``` cpp
void UGameUIManagerSubsystem::NotifyPlayerAdded(UCommonLocalPlayer* LocalPlayer)
{
	if (ensure(LocalPlayer) && CurrentPolicy)
	{
		CurrentPolicy->NotifyPlayerAdded(LocalPlayer);
	}
}
```
### 代码

#### UGameUIManagerSubsystem
``` cpp

/**
 * This manager is intended to be replaced by whatever your game needs to
 * actually create, so this class is abstract to prevent it from being created.
 *
 * 这个管理器旨在根据您的游戏实际需求进行替换，因此这个类是抽象类，以防止其被创建。
 * 
 * If you just need the basic functionality you will start by sublcassing this
 * subsystem in your own game.
 *
 * 如果您只需要基本功能，那么您只需在自己的游戏中继承此子系统即可。
 * 
 */
UCLASS(MinimalAPI, Abstract, config = Game)
class UGameUIManagerSubsystem : public UGameInstanceSubsystem
{
	GENERATED_BODY()
	
public:
	UGameUIManagerSubsystem() { }

	// 创建策略
	UE_API virtual void Initialize(FSubsystemCollectionBase& Collection) override;

	// 回收策略
	UE_API virtual void Deinitialize() override;

	// 如果开发者没有自定义才需要创建该子系统
	UE_API virtual bool ShouldCreateSubsystem(UObject* Outer) const override;

	// 获取当前策略
	const UGameUIPolicy* GetCurrentUIPolicy() const { return CurrentPolicy; }

	// 获取当前策略
	UGameUIPolicy* GetCurrentUIPolicy() { return CurrentPolicy; }

	// 转发
	UE_API virtual void NotifyPlayerAdded(UCommonLocalPlayer* LocalPlayer);

	// 转发
	UE_API virtual void NotifyPlayerRemoved(UCommonLocalPlayer* LocalPlayer);

	// 转发
	UE_API virtual void NotifyPlayerDestroyed(UCommonLocalPlayer* LocalPlayer);

protected:
	// 切换策略
	UE_API void SwitchToPolicy(UGameUIPolicy* InPolicy);

private:
	// 持有
	UPROPERTY(Transient)
	TObjectPtr<UGameUIPolicy> CurrentPolicy = nullptr;

	// 配置
	UPROPERTY(config, EditAnywhere)
	TSoftClassPtr<UGameUIPolicy> DefaultUIPolicyClass;
};

#undef UE_API

```


#### ULyraUIManagerSubsystem
``` cpp

UCLASS()
class ULyraUIManagerSubsystem : public UGameUIManagerSubsystem
{
	GENERATED_BODY()

public:

	ULyraUIManagerSubsystem();

	// 绑定Tick
	virtual void Initialize(FSubsystemCollectionBase& Collection) override;

	// 移除Tick
	virtual void Deinitialize() override;

private:
	// Tick
	bool Tick(float DeltaTime);
	// 更改UI布局的可实现
	void SyncRootLayoutVisibilityToShowHUD();
	
	FTSTicker::FDelegateHandle TickHandle;
};

```

## 配置加载界面
DefaultGame.ini
```ini

[/Script/CommonLoadingScreen.CommonLoadingScreenSettings]
; 默认的加载界面
LoadingScreenWidget=/Game/UI/Foundation/LoadingScreen/W_LoadingScreen_Host.W_LoadingScreen_Host_C
; 强制Tick加载界面即使是在编辑器
ForceTickLoadingScreenEvenInEditor=False

```
注意此时该蓝图界面有逻辑报错.直接删除即可.我们只需要先临时使用控件即可.

如果没配置 蓝图运行会提示日志报错
``` ini
PIE: Error: CreateWidget called with a null class.
LogLoadingScreen: Error: Failed to load the loading screen widget , falling back to placeholder.
```



## 总结
因为我们需要测试Experience及后续加载流程.所以本节先临时修复一下UI.后续章节再详细展开.
此时还有很多蓝图资产报错,可以不理会.自行删掉也行.