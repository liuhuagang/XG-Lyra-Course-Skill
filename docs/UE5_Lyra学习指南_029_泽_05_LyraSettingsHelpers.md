# UE5_Lyra学习指南_029_泽_05_LyraSettingsHelpers

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_029\_泽\_05\_LyraSettingsHelpers](#ue5_lyra学习指南_029_泽_05_lyrasettingshelpers)
	- [概述](#概述)
	- [TMobileQualityWrapper](#tmobilequalitywrapper)
	- [LyraSettingsHelpers](#lyrasettingshelpers)
		- [声明监听的命令行变量](#声明监听的命令行变量)
		- [工具方法](#工具方法)
	- [总结](#总结)



## 概述
本节主要讲解LyraSettingsLocal中的辅助方法和结构体.

## TMobileQualityWrapper
这个类主要用于缓存数据,便于查询更改.
``` cpp

//////////////////////////////////////////////////////////////////////
// 移动端质量包装
template <typename T>
struct TMobileQualityWrapper
{
private:
	// 默认值
	T DefaultValue;
	// 关联的命令行变量
	TAutoConsoleVariable<FString>& WatchedVar;
	// 上次关联的命令行变量观测到的值
	FString LastSeenCVarString;

	// 阈值的键值对
	struct FLimitPair
	{
		int32 Limit = 0;
		T Value = T(0);
	};

	// 阈值键值对的容器
	TArray<FLimitPair> Thresholds;

public:
	// 构造函数 指定默认值 关联命令行变量 必须是字符串类型
	TMobileQualityWrapper(T InDefaultValue, TAutoConsoleVariable<FString>& InWatchedVar)
		: DefaultValue(InDefaultValue)
		  , WatchedVar(InWatchedVar)
	{
	}

	// 传入键值 返回符合要求的值 否则就返回默认值
	T Query(int32 TestValue)
	{
		// 更新缓存 获取到最新的值
		UpdateCache();

		for (const FLimitPair& Pair : Thresholds)
		{
			if (TestValue >= Pair.Limit)
			{
				return Pair.Value;
			}
		}

		return DefaultValue;
	}

	// Returns the first threshold value or INDEX_NONE if there aren't any
	// 返回第一个阈值值，若没有则返回 INDEX_NONE
	int32 GetFirstThreshold()
	{
		UpdateCache();
		return (Thresholds.Num() > 0) ? Thresholds[0].Limit : INDEX_NONE;
	}

	// Returns the lowest value of all the pairs or DefaultIfNoPairs if there are no pairs
	// 返回所有配对项中的最小值；若无配对项，则返回“默认值（若无默认值则返回空值）”
	T GetLowestValue(T DefaultIfNoPairs)
	{
		UpdateCache();

		T Result = DefaultIfNoPairs;
		bool bFirstValue = true;
		for (const FLimitPair& Pair : Thresholds)
		{
			if (bFirstValue)
			{
				Result = Pair.Value;
				bFirstValue = false;
			}
			else
			{
				Result = FMath::Min(Result, Pair.Value);
			}
		}

		return Result;
	}

private:
	// 更新缓存
	void UpdateCache()
	{
		// 获取到最新的字符串
		const FString CurrentValue = WatchedVar.GetValueOnGameThread();

		// 应该与之前的字符串不一致
		if (!CurrentValue.Equals(LastSeenCVarString, ESearchCase::CaseSensitive))
		{
			// 存储此次的值作为前值
			LastSeenCVarString = CurrentValue;

			// 重置容器
			Thresholds.Reset();

			// Parse the thresholds
			// 切割阈值
			int32 ScanIndex = 0;
			
			while (ScanIndex < LastSeenCVarString.Len())
			{
				const int32 ColonIndex = LastSeenCVarString.Find(
					TEXT(":"), ESearchCase::CaseSensitive, ESearchDir::FromStart, ScanIndex);
				
				if (ColonIndex > 0)
				{
					const int32 CommaIndex = LastSeenCVarString.Find(
						TEXT(","), ESearchCase::CaseSensitive, ESearchDir::FromStart, ColonIndex);
					const int32 EndOfPairIndex = (CommaIndex != INDEX_NONE) ? CommaIndex : LastSeenCVarString.Len();

					FLimitPair Pair;
					LexFromString(Pair.Limit, *LastSeenCVarString.Mid(ScanIndex, ColonIndex - ScanIndex));
					LexFromString(Pair.Value, *LastSeenCVarString.Mid(ColonIndex + 1, EndOfPairIndex - ColonIndex - 1));
					Thresholds.Add(Pair);

					ScanIndex = EndOfPairIndex + 1;
				}
				else
				{
					UE_LOG(LogConsoleResponse, Error, TEXT("Malformed value for '%s'='%s', expecting a ':'"),
					       *IConsoleManager::Get().FindConsoleObjectName(WatchedVar.AsVariable()),
					       *LastSeenCVarString);
					Thresholds.Reset();
					break;
				}
			}

			// Sort the pairs
			// 排序
			Thresholds.Sort([](const FLimitPair A, const FLimitPair B) { return A.Limit < B.Limit; });
		}
	}
};

```

## LyraSettingsHelpers

### 声明监听的命令行变量
``` cpp
	//整体的质量限制默认为-1,监听的命令字符串用于动态切割CVarMobileQualityLimits
	TMobileQualityWrapper<int32> OverallQualityLimits(-1, CVarMobileQualityLimits);

	//整体的质量限制默认为-1,监听的命令字符串用于动态切割CVarMobileQualityLimits
	TMobileQualityWrapper<float> ResolutionQualityLimits(100.0f, CVarMobileResolutionQualityLimits);

	//整体的质量限制默认为-1,监听的命令字符串用于动态切割CVarMobileQualityLimits
	TMobileQualityWrapper<float> ResolutionQualityRecommendations(75.0f, CVarMobileResolutionQualityRecommendation);
```
注意这三兄弟都是FString类型.
``` cpp
//////////////////////////////////////////////////////////////////////
// “关于分辨率质量的限制列表，格式为“帧率：最高质量，帧率2：最高质量2……”，当帧率达到或超过阈值时即生效”
static TAutoConsoleVariable<FString> CVarMobileQualityLimits(
	TEXT("Lyra.DeviceProfile.Mobile.OverallQualityLimits"),
	TEXT(""),
	TEXT(
		"List of limits on resolution quality of the form \"FPS:MaxQuality,FPS2:MaxQuality2,...\", kicking in when FPS is at or above the threshold"),
	ECVF_Default | ECVF_Preview);
// “关于分辨率质量的限制列表，格式为“帧率：最高分辨率质量，帧率2：最高分辨率质量2，……”，当帧率达到或超过阈值时即生效”
static TAutoConsoleVariable<FString> CVarMobileResolutionQualityLimits(
	TEXT("Lyra.DeviceProfile.Mobile.ResolutionQualityLimits"),
	TEXT(""),
	TEXT(
		"List of limits on resolution quality of the form \"FPS:MaxResQuality,FPS2:MaxResQuality2,...\", kicking in when FPS is at or above the threshold"),
	ECVF_Default | ECVF_Preview);
// “关于分辨率质量的限制列表，格式为“帧率：建议值，帧率2：建议值2……”，当帧率达到或超过阈值时即生效”
static TAutoConsoleVariable<FString> CVarMobileResolutionQualityRecommendation(
	TEXT("Lyra.DeviceProfile.Mobile.ResolutionQualityRecommendation"),
	TEXT("0:75"),
	TEXT(
		"List of limits on resolution quality of the form \"FPS:Recommendation,FPS2:Recommendation2,...\", kicking in when FPS is at or above the threshold"),
	ECVF_Default | ECVF_Preview);

//////////////////////////////////////////////////////////////////////

```


### 工具方法
``` cpp

namespace LyraSettingsHelpers
{
	// 检测当前平台特性
	bool HasPlatformTrait(FGameplayTag Tag)
	{
		return ICommonUIModule::GetSettings().GetPlatformTraits().HasTag(Tag);
	}

	// Returns the max level from the integer scalability settings (ignores ResolutionQuality)
	// 返回整数可扩展性设置中的最大级别（忽略分辨率质量）
	int32 GetHighestLevelOfAnyScalabilityChannel(const Scalability::FQualityLevels& ScalabilityQuality)
	{
		static_assert(sizeof(Scalability::FQualityLevels) == 88,
		              "This function may need to be updated to account for new members");

		int32 MaxScalability = ScalabilityQuality.ViewDistanceQuality;
		MaxScalability = FMath::Max(MaxScalability, ScalabilityQuality.AntiAliasingQuality);
		MaxScalability = FMath::Max(MaxScalability, ScalabilityQuality.ShadowQuality);
		MaxScalability = FMath::Max(MaxScalability, ScalabilityQuality.GlobalIlluminationQuality);
		MaxScalability = FMath::Max(MaxScalability, ScalabilityQuality.ReflectionQuality);
		MaxScalability = FMath::Max(MaxScalability, ScalabilityQuality.PostProcessQuality);
		MaxScalability = FMath::Max(MaxScalability, ScalabilityQuality.TextureQuality);
		MaxScalability = FMath::Max(MaxScalability, ScalabilityQuality.EffectsQuality);
		MaxScalability = FMath::Max(MaxScalability, ScalabilityQuality.FoliageQuality);
		MaxScalability = FMath::Max(MaxScalability, ScalabilityQuality.ShadingQuality);

		return (MaxScalability >= 0) ? MaxScalability : -1;
	}

	// 从设备文件配置拓展性能设置
	void FillScalabilitySettingsFromDeviceProfile(FLyraScalabilitySnapshot& Mode, const FString& Suffix = FString())
	{
		static_assert(sizeof(Scalability::FQualityLevels) == 88,
		              "This function may need to be updated to account for new members");

		// Default out before filling so we can correctly mark non-overridden scalability values.
		// It's technically possible to swap device profile when testing so safest to clear and refill

		// 在填充数据之前先进行默认输出，这样我们就能准确地标记未被覆盖的可扩展性值。
		// 在测试时确实有可能更换设备配置文件，因此最安全的做法是先清空再重新填充。
		Mode = FLyraScalabilitySnapshot();

		Mode.bHasOverrides |= UDeviceProfileManager::GetScalabilityCVar(
			FString::Printf(TEXT("sg.ResolutionQuality%s"), *Suffix), Mode.Qualities.ResolutionQuality);
		Mode.bHasOverrides |= UDeviceProfileManager::GetScalabilityCVar(
			FString::Printf(TEXT("sg.ViewDistanceQuality%s"), *Suffix), Mode.Qualities.ViewDistanceQuality);
		Mode.bHasOverrides |= UDeviceProfileManager::GetScalabilityCVar(
			FString::Printf(TEXT("sg.AntiAliasingQuality%s"), *Suffix), Mode.Qualities.AntiAliasingQuality);
		Mode.bHasOverrides |= UDeviceProfileManager::GetScalabilityCVar(
			FString::Printf(TEXT("sg.ShadowQuality%s"), *Suffix), Mode.Qualities.ShadowQuality);
		Mode.bHasOverrides |= UDeviceProfileManager::GetScalabilityCVar(
			FString::Printf(TEXT("sg.GlobalIlluminationQuality%s"), *Suffix), Mode.Qualities.GlobalIlluminationQuality);
		Mode.bHasOverrides |= UDeviceProfileManager::GetScalabilityCVar(
			FString::Printf(TEXT("sg.ReflectionQuality%s"), *Suffix), Mode.Qualities.ReflectionQuality);
		Mode.bHasOverrides |= UDeviceProfileManager::GetScalabilityCVar(
			FString::Printf(TEXT("sg.PostProcessQuality%s"), *Suffix), Mode.Qualities.PostProcessQuality);
		Mode.bHasOverrides |= UDeviceProfileManager::GetScalabilityCVar(
			FString::Printf(TEXT("sg.TextureQuality%s"), *Suffix), Mode.Qualities.TextureQuality);
		Mode.bHasOverrides |= UDeviceProfileManager::GetScalabilityCVar(
			FString::Printf(TEXT("sg.EffectsQuality%s"), *Suffix), Mode.Qualities.EffectsQuality);
		Mode.bHasOverrides |= UDeviceProfileManager::GetScalabilityCVar(
			FString::Printf(TEXT("sg.FoliageQuality%s"), *Suffix), Mode.Qualities.FoliageQuality);
		Mode.bHasOverrides |= UDeviceProfileManager::GetScalabilityCVar(
			FString::Printf(TEXT("sg.ShadingQuality%s"), *Suffix), Mode.Qualities.ShadingQuality);
	}

	TMobileQualityWrapper<int32> OverallQualityLimits(-1, CVarMobileQualityLimits);
	TMobileQualityWrapper<float> ResolutionQualityLimits(100.0f, CVarMobileResolutionQualityLimits);
	TMobileQualityWrapper<float> ResolutionQualityRecommendations(75.0f, CVarMobileResolutionQualityRecommendation);

	// 获取该对应帧数的画面质量限制
	int32 GetApplicableOverallQualityLimit(int32 FrameRate)
	{
		return OverallQualityLimits.Query(FrameRate);
	}
	// 获取该对应帧数的画面分辨率质量限制
	float GetApplicableResolutionQualityLimit(int32 FrameRate)
	{
		return ResolutionQualityLimits.Query(FrameRate);
	}

	// 获取该对应帧数的建议画面分辨率质量限制
	float GetApplicableResolutionQualityRecommendation(int32 FrameRate)
	{
		return ResolutionQualityRecommendations.Query(FrameRate);
	}

	// 约束帧数,通过兼容的画面质量
	int32 ConstrainFrameRateToBeCompatibleWithOverallQuality(int32 FrameRate, int32 OverallQuality)
	{
		const ULyraPlatformSpecificRenderingSettings* PlatformSettings = ULyraPlatformSpecificRenderingSettings::Get();
		const TArray<int32>& PossibleRates = PlatformSettings->MobileFrameRateLimits;

		// Choose the closest frame rate (without going over) to the user preferred one that is supported and compatible with the desired overall quality
		// 选择与用户偏好帧率最为接近（且不超过该值）的帧率，该帧率需支持且与预期的整体画质相兼容。
		int32 LimitIndex = PossibleRates.FindLastByPredicate([=](const int32& TestRate)
		{
			// 帧数用户值必须高于等于我们的设定值
			const bool bAtOrBelowDesiredRate = (TestRate <= FrameRate);

			// 获取我们的设定值对应的画面质量
			const int32 LimitQuality = GetApplicableResolutionQualityLimit(TestRate);

			// 我们的设定的画面之必须有效
			// 我们设定的画面质量必须高于等于用户的需求质量
			const bool bQualityDoesntExceedLimit = (LimitQuality < 0) || (OverallQuality <= LimitQuality);

			// 我们的设定的帧数必须项目支持且硬件平台支持
			const bool bIsSupported = ULyraSettingsLocal::IsSupportedMobileFramePace(TestRate);

			// 三者都满足的情况下.这组条件可以条件可以使用.
			return bAtOrBelowDesiredRate && bQualityDoesntExceedLimit && bIsSupported;
		});

		return PossibleRates.IsValidIndex(LimitIndex)
			       ? PossibleRates[LimitIndex]
			       : ULyraSettingsLocal::GetDefaultMobileFrameRate();
	}

	// Returns the first frame rate at which overall quality is restricted/limited by the current device profile
	// 返回当前设备配置所限制/约束整体质量的首个帧率值
	int32 GetFirstFrameRateWithQualityLimit()
	{
		return OverallQualityLimits.GetFirstThreshold();
	}

	// Returns the lowest quality at which there's a limit on the overall frame rate (or -1 if there is no limit)
	// 返回整体帧率存在上限的最低质量等级（若不存在上限，则返回 -1）
	int32 GetLowestQualityWithFrameRateLimit()
	{
		return OverallQualityLimits.GetLowestValue(-1);
	}
}

//////////////////////////////////////////////////////////////////////

```

## 总结
本节主要介绍了本地设置的一些辅助方法.避免设置类和其他配置类耦合过多.