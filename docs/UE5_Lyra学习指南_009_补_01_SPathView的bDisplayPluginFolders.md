# UE5_Lyra学习指南_009_补_01_SPathView的bDisplayPluginFolders

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.  
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)  
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)  

文章内容由小刚撰写,采用了以下多种方式:  
1.口述转文字  
2.AI重构  
3.参考引擎源码  
4.Lyra工程源码  
5.结合社区论坛各位大佬的解析  

- [UE5\_Lyra学习指南\_009\_补\_01\_SPathView的bDisplayPluginFolders](#ue5_lyra学习指南_009_补_01_spathview的bdisplaypluginfolders)
	- [概述](#概述)
	- [5.6](#56)
	- [5.0](#50)
	- [调用位置](#调用位置)
	- [总结](#总结)



## 概述
Lyra项目本身是由5.0代码开发完成,并随着引擎版本进行小的修修补补.
一般来说,没有编译报错或者影响主干游戏功能,它就不会系统性地维护.
## 5.6
``` cpp

FContentBrowserDataCompiledFilter SPathView::CreateCompiledFolderFilter() const
{
	UE_LOG(LogPathView, VeryVerbose, TEXT("[%s] Creating folder filter"), *WriteToString<256>(OwningContentBrowserName));

	const UContentBrowserSettings* ContentBrowserSettings = GetDefault<UContentBrowserSettings>();
	bool bDisplayPluginFolders = ContentBrowserSettings->GetDisplayPluginFolders();
	// check to see if we have an instance config that overrides the default in UContentBrowserSettings
	if (FContentBrowserInstanceConfig* EditorConfig = GetContentBrowserConfig())
	{
		bDisplayPluginFolders = EditorConfig->bShowPluginContent;
	}

	FContentBrowserDataFilter DataFilter;
	DataFilter.bRecursivePaths = true;
	DataFilter.ItemTypeFilter = EContentBrowserItemTypeFilter::IncludeFolders;
	DataFilter.ItemCategoryFilter = GetContentBrowserItemCategoryFilter();
	DataFilter.ItemAttributeFilter = GetContentBrowserItemAttributeFilter();

	UE_LOG(LogPathView, VeryVerbose, TEXT("[%s] bDisplayPluginFolders:%d ItemCategoryFilter:%d ItemAttributeFilter:%d"), 
		*WriteToString<256>(OwningContentBrowserName), bDisplayPluginFolders, DataFilter.ItemCategoryFilter, DataFilter.ItemAttributeFilter);

	TSharedPtr<FPathPermissionList> CombinedFolderPermissionList = ContentBrowserUtils::GetCombinedFolderPermissionList(FolderPermissionList, bAllowReadOnlyFolders ? nullptr : WritableFolderPermissionList);

	if (CustomFolderPermissionList.IsValid())
	{
		if (!CombinedFolderPermissionList.IsValid())
		{
			CombinedFolderPermissionList = MakeShared<FPathPermissionList>();
		}
		CombinedFolderPermissionList->Append(*CustomFolderPermissionList);
	}

	if (PluginPathFilters.IsValid() && PluginPathFilters->Num() > 0 && bDisplayPluginFolders)
	{
		UE_SUPPRESS(LogPathView, VeryVerbose, {
			FString PluginFiltersString;
			for (int32 i=0; i < PluginPathFilters->Num(); ++i)
			{
				if (i != 0)
				{
					PluginFiltersString += TEXT(", ");
				}
				PluginFiltersString += PluginPathFilters->GetFilterAtIndex(i)->GetName();
			}
			UE_LOG(LogPathView, VeryVerbose, TEXT("[%s] Active plugin filters: %s"), 
				*WriteToString<256>(OwningContentBrowserName), *PluginFiltersString);
		});
		TArray<TSharedRef<IPlugin>> Plugins = IPluginManager::Get().GetEnabledPluginsWithContent();
		for (const TSharedRef<IPlugin>& Plugin : Plugins)
		{
			if (!PluginPathFilters->PassesAllFilters(Plugin))
			{
				FString MountedAssetPath = Plugin->GetMountedAssetPath();
				MountedAssetPath.RemoveFromEnd(TEXT("/"), ESearchCase::CaseSensitive);

				if (!CombinedFolderPermissionList.IsValid())
				{
					CombinedFolderPermissionList = MakeShared<FPathPermissionList>();
				}
				CombinedFolderPermissionList->AddDenyListItem("PluginPathFilters", MountedAssetPath);
			}
		}
	}

	UE_LOG(LogPathView, VeryVerbose, TEXT("Compiled folder permission list: %s"), CombinedFolderPermissionList.IsValid() ? *CombinedFolderPermissionList->ToString() : TEXT("null"));

	ContentBrowserUtils::AppendAssetFilterToContentBrowserFilter(FARFilter(), nullptr, CombinedFolderPermissionList, DataFilter);

	FContentBrowserDataCompiledFilter CompiledDataFilter;
	{
		static const FName RootPath = "/";
		UContentBrowserDataSubsystem* ContentBrowserData = IContentBrowserDataModule::Get().GetSubsystem();
		ContentBrowserData->CompileFilter(RootPath, DataFilter, CompiledDataFilter);
	}
	return CompiledDataFilter;
}

```
## 5.0
``` cpp

FContentBrowserDataCompiledFilter SPathView::CreateCompiledFolderFilter() const
{
	const UContentBrowserSettings* ContentBrowserSettings = GetDefault<UContentBrowserSettings>();

	FContentBrowserDataFilter DataFilter;
	DataFilter.bRecursivePaths = true;
	DataFilter.ItemTypeFilter = EContentBrowserItemTypeFilter::IncludeFolders;
	DataFilter.ItemCategoryFilter = GetContentBrowserItemCategoryFilter();
	DataFilter.ItemAttributeFilter = GetContentBrowserItemAttributeFilter();

	TSharedPtr<FPathPermissionList> CombinedFolderPermissionList = ContentBrowserUtils::GetCombinedFolderPermissionList(FolderPermissionList, bAllowReadOnlyFolders ? nullptr : WritableFolderPermissionList);

	if (CustomFolderPermissionList.IsValid())
	{
		if (!CombinedFolderPermissionList.IsValid())
		{
			CombinedFolderPermissionList = MakeShared<FPathPermissionList>();
		}
		CombinedFolderPermissionList->Append(*CustomFolderPermissionList);
	}

	if (PluginPathFilters.IsValid() && PluginPathFilters->Num() > 0 && ContentBrowserSettings->GetDisplayPluginFolders())
	{
		TArray<TSharedRef<IPlugin>> Plugins = IPluginManager::Get().GetEnabledPluginsWithContent();
		for (const TSharedRef<IPlugin>& Plugin : Plugins)
		{
			if (!PluginPathFilters->PassesAllFilters(Plugin))
			{
				FString MountedAssetPath = Plugin->GetMountedAssetPath();
				MountedAssetPath.RemoveFromEnd(TEXT("/"), ESearchCase::CaseSensitive);

				if (!CombinedFolderPermissionList.IsValid())
				{
					CombinedFolderPermissionList = MakeShared<FPathPermissionList>();
				}
				CombinedFolderPermissionList->AddDenyListItem("PluginPathFilters", MountedAssetPath);
			}
		}
	}

	ContentBrowserUtils::AppendAssetFilterToContentBrowserFilter(FARFilter(), nullptr, CombinedFolderPermissionList, DataFilter);

	FContentBrowserDataCompiledFilter CompiledDataFilter;
	{
		static const FName RootPath = "/";
		UContentBrowserDataSubsystem* ContentBrowserData = IContentBrowserDataModule::Get().GetSubsystem();
		ContentBrowserData->CompileFilter(RootPath, DataFilter, CompiledDataFilter);
	}
	return CompiledDataFilter;
}

```
## 调用位置
``` cpp
void ULyraEditorEngine::FirstTickSetup()
{
	if (bFirstTickSetup)
	{
		return;
	}

	bFirstTickSetup = true;

	// Force show plugin content on load.
	GetMutableDefault<UContentBrowserSettings>()->SetDisplayPluginFolders(true);

}

```

## 总结
这个位置主要让大家知道可以嵌入一些自定义代码