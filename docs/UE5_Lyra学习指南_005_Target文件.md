# UE5_Lyra学习指南_005_Target文件

本文章仅为小刚-B站课堂-虚幻引擎视频课程Lyra-精讲的演讲手稿.
本套课程链接:[[UE5]虚幻引擎游戏案例Lyra精讲](https://www.bilibili.com/cheese/play/ss112001159)
前置课程链接:[[UE5]虚幻引擎UEC++从基础到进阶](https://www.bilibili.com/cheese/play/ss28043)

文章内容由小刚撰写,采用了以下多种方式:

1.口述转文字
2.AI重构
3.参考引擎源码
4.Lyra工程源码
5.结合社区论坛各位大佬的解析

可能存在内容不是完全准确,请以视频内容呈现为最终效果.
- [UE5\_Lyra学习指南\_005\_Target文件](#ue5_lyra学习指南_005_target文件)
	- [创建4个Target.cs文件](#创建4个targetcs文件)
	- [完善LyraGameTarget的设置方法](#完善lyragametarget的设置方法)
	- [总结](#总结)


## 创建4个Target.cs文件

本节只完善.Target.cs文件,.Build.cs将在下节完善.

请依次创建:
1.LyraGame.Target.cs
2.LyraEditor.Target.cs
3.LyraClient.Target.cs
4.LyraServer.Target.cs

分别设置TargetType为Game,Editor,Client,Server.

## 完善LyraGameTarget的设置方法
Client
为了方便统一设置,在LyraGameTarget中创建名为internal static void ApplySharedLyraTargetSettings(TargetRules Target)的方法.

LyraGameTarget.target.cs文件已经详细注释.此处直接贴出代码.注意,包含使用到的头文件.
``` cs
// Copyright Epic Games, Inc. All Rights Reserved.

using UnrealBuildTool;
using System;
using System.IO;
using EpicGames.Core;
using System.Collections.Generic;
using UnrealBuildBase;
using Microsoft.Extensions.Logging;

public class LyraGameTarget : TargetRules
{
	public LyraGameTarget(TargetInfo Target) : base(Target)
	{
		Type = TargetType.Game;
	
		// List of additional modules to be compiled into the target.
		// 用于编译到目标程序中的附加模块列表。
		
		ExtraModuleNames.Add("LyraGame");
		
		LyraGameTarget.ApplySharedLyraTargetSettings(this);
	}
	
	private static bool bHasWarnedAboutShared = false;
	internal static void ApplySharedLyraTargetSettings(TargetRules Target)
	{
		// Logger for output relating to this target. Set before the constructor is run from RulesCompiler
		// 用于记录与该目标相关数据的日志。在从规则编译器运行构造函数之前进行设置。
		ILogger Logger = Target.Logger;
		
		// Specifies the engine version to maintain backwards-compatible default build settings with (eg. DefaultSettingsVersion.Release_4_23, DefaultSettingsVersion.Release_4_24). Specify DefaultSettingsVersion.Latest to always use defaults for the current engine version, at the risk of introducing build errors while upgrading.
		// 指定要维持与旧版本兼容的默认构建设置的引擎版本（例如：DefaultSettingsVersion.Release_4_23、DefaultSettingsVersion.Release_4_24）。指定 DefaultSettingsVersion.Latest 会始终使用当前引擎版本的默认设置，但这样做可能会在升级过程中引入构建错误。
		Target.DefaultBuildSettings = BuildSettingsVersion.V5;
		
		// What version of include order to use when compiling this target. Can be overridden via -ForceIncludeOrder on the command line. ModuleRules.IncludeOrderVersion takes precedence.
		// 在编译此目标时应使用何种包含顺序。可通过命令行中的 -ForceIncludeOrder 参数进行覆盖。ModuleRules.IncludeOrderVersion 具有优先级。
		Target.IncludeOrderVersion = EngineIncludeOrderVersion.Unreal5_6;
		
		// Test configuration
		// 测试配置
		bool bIsTest = Target.Configuration == UnrealTargetConfiguration.Test;
		
		// Shipping configuration
		// 发型配置
		bool bIsShipping = Target.Configuration == UnrealTargetConfiguration.Shipping;
		
		// Alias for TargetType.Server
		// 专属服务器
		bool bIsDedicatedServer = Target.Type == TargetType.Server;
		
		//Engine binaries and intermediates are specific to this target
		//引擎的二进制文件和中间文件是针对此目标特定的。
		if (Target.BuildEnvironment == TargetBuildEnvironment.Unique)
		{
			// C++ Warnings settings object used by the target
			// 由目标程序所使用的 C++ 警告设置对象
			
			// Forces shadow variable warnings to be treated as errors on platforms that support it. MSVC - https://learn.microsoft.com/en-us/cpp/error-messages/compiler-warnings/compiler-warning-level-4-c4456 4456 - declaration of 'LocalVariable' hides previous local declaration https://learn.microsoft.com/en-us/cpp/error-messages/compiler-warnings/compiler-warning-level-4-c4458 4458 - declaration of 'parameter' hides class member https://learn.microsoft.com/en-us/cpp/error-messages/compiler-warnings/compiler-warning-level-4-c4459 4459 - declaration of 'LocalVariable' hides global declaration Clang - https://clang.llvm.org/docs/DiagnosticsReference.html#wshadow
			// 对于支持该功能的平台，会将“阴影变量警告”视为错误。MSVC - https://learn.microsoft.com/en-us/cpp/error-messages/compiler-warnings/compiler-warning-level-4-c4456 4456 - “局部变量”的声明遮蔽了之前的局部声明 https://learn.microsoft.com/en-us/cpp/error-messages/compiler-warnings/compiler-warning-level-4-c4458 4458 - “参数”的声明遮蔽了类成员 https://learn.microsoft.com/en-us/cpp/error-messages/compiler-warnings/compiler-warning-level-4-c4459 4459 - “局部变量”的声明遮蔽了全局声明 Clang - https://clang.llvm.org/docs/DiagnosticsReference.html#wshadow
			
			// Output warnings as errors
			// 输出的警告级别为错误
			Target.CppCompileWarningSettings.ShadowVariableWarningLevel = WarningLevel.Error;
			
			// Whether to turn on logging for test/shipping builds.
			// 是否开启测试/发布版本的日志记录功能。
			Target.bUseLoggingInShipping = true;
			
			// Whether to track owner (asset name) of RHI resource for Test configuration. Useful for ListShaderMaps and ListShaderLibraries commands.
			// 是否在测试配置中跟踪 RHI 资源的所有者（资产名称）。此功能对“列表着色器映射”和“列表着色器库”命令非常有用。
			Target.bTrackRHIResourceInfoForTest = true;
			
			//发行模式且非专属服务器
			if (bIsShipping && !bIsDedicatedServer)
			{
				// Make sure that we validate certificates for HTTPS traffic
				// 请确保对用于 HTTPS 流量的证书进行验证
				
				// Whether to allow engine configuration to determine if we can load unverified certificates.
				// 是否允许通过引擎配置来决定我们是否能够加载未经验证的证书。
				Target.bDisableUnverifiedCertificates = true;

				// Uncomment these lines to lock down the command line processing
				// 取消这些行的注释即可锁定命令行处理过程
				// This will only allow the specified command line arguments to be parsed
				// 这将仅允许对指定的命令行参数进行解析。
				//Target.GlobalDefinitions.Add("UE_COMMAND_LINE_USES_ALLOW_LIST=1");
				//Target.GlobalDefinitions.Add("UE_OVERRIDE_COMMAND_LINE_ALLOW_LIST=\"-space -separated -list -of -commands\"");

				// Uncomment this line to filter out sensitive command line arguments that you
				// 取消注释这一行，以便过滤掉那些敏感的命令行参数。您需要这样做。
				// don't want to go into the log file (e.g., if you were uploading logs)
				// 不想查看日志文件（例如，如果您正在上传日志的话）
				//Target.GlobalDefinitions.Add("FILTER_COMMANDLINE_LOGGING=\"-some_connection_id -some_other_arg\"");
			}
			//发行模式或者测试模式下
			if (bIsShipping || bIsTest)
			{
				// Disable reading generated/non-ufs ini files
				// 禁用读取生成的/非 UFS 格式的 ini 文件
				
				// Whether to load generated ini files in cooked build, (GameUserSettings.ini loaded either way)
				// 是否在烘焙版本中加载生成的 ini 文件？（无论采用哪种方式加载，GameUserSettings.ini 都会被加载）
				Target.bAllowGeneratedIniWhenCooked = false;
				
				// Whether to load non-ufs ini files in cooked build, (GameUserSettings.ini loaded either way)
				// 在烘焙版本中是否要加载非 ufs 格式的 ini 文件？（无论采用哪种方式加载，GameUserSettings.ini 都会被加载）
				Target.bAllowNonUFSIniWhenCooked = false;
			}
			//非编辑器模式下
			if (Target.Type != TargetType.Editor)
			{
				// We don't use the path tracer at runtime, only for beauty shots, and this DLL is quite large
				// 我们在运行时并不使用路径追踪功能，仅用于制作精美的效果图，而且这个动态链接库的体积相当大。
				Target.DisablePlugins.Add("OpenImageDenoise");

				// Reduce memory use in AssetRegistry always-loaded data, but add more cputime expensive queries
				// 减少资产注册表中始终加载数据所占用的内存，但增加一些计算资源消耗较大的查询操作
				Target.GlobalDefinitions.Add("UE_ASSETREGISTRY_INDIRECT_ASSETDATA_POINTERS=1");
			}

			LyraGameTarget.ConfigureGameFeaturePlugins(Target);
		}
		else
		{
			// !!!!!!!!!!!! WARNING !!!!!!!!!!!!!
			// Any changes in here must not affect PCH generation, or the target needs to be set to TargetBuildEnvironment.Unique
			
			// !!!!!!!!!!!! 警告 ！！！！！！！！！！！！！
			// 此处的任何修改都不得影响预编译头文件的生成，否则目标环境必须设置为TargetBuildEnvironment.Unique
			
			// This only works in editor or Unique build environments
			// 仅在编辑器或独占式构建环境中有效
			if (Target.Type == TargetType.Editor)
			{
				LyraGameTarget.ConfigureGameFeaturePlugins(Target);
			}
			else
			{
				// Shared monolithic builds cannot enable/disable plugins or change any options because it tries to re-use the installed engine binaries
				// 共享的单体式构建无法启停插件或更改任何选项，因为其会尝试复用已安装的引擎二进制文件。
				if (!bHasWarnedAboutShared)
				{
					bHasWarnedAboutShared = true;
					Logger.LogWarning("LyraGameEOS and dynamic target options are disabled when packaging from an installed version of the engine");
				}
			}
		}
	
	}
	
	
	static public bool ShouldEnableAllGameFeaturePlugins(TargetRules Target)
	{
		if (Target.Type == TargetType.Editor)
		{
			// With return true, editor builds will build all game feature plugins, but it may or may not load them all.
			// This is so you can enable plugins in the editor without needing to compile code.
			
			// 若设置为“true”，编辑器将构建所有游戏功能插件，但这些插件是否会被全部加载则取决于具体情况。
			// 这样您就可以在编辑器中启用插件，而无需编译代码。
			
			// return true;
		}

		bool bIsBuildMachine = (Environment.GetEnvironmentVariable("IsBuildMachine") == "1");
		
		if (bIsBuildMachine)
		{
			// This could be used to enable all plugins for build machines
			// 这可以用于为构建机器启用所有插件
			// return true;
		}

		// By default use the default plugin rules as set by the plugin browser in the editor
		// This is important because this code may not be run at all for launcher-installed versions of the engine
		
		// 默认情况下，将使用插件浏览器在编辑器中设置的默认插件规则
		// 这非常重要，因为对于安装在启动器中的引擎版本，这段代码可能根本不会被执行
		return false;
	}
	
	//一个插件名和它的JsonObject的对象,用于获取插件描述信息
	private static Dictionary<string, JsonObject> AllPluginRootJsonObjectsByName = new Dictionary<string, JsonObject>();
	
	// Configures which game feature plugins we want to have enabled
	// This is a fairly simple implementation, but you might do things like build different
	// plugins based on the target release version of the current branch, e.g., enabling 
	// work-in-progress features in main but disabling them in the current release branch.
	
	// 用于配置我们希望启用哪些游戏功能插件
	// 这是一种相对简单的实现方式，但您也可以根据当前分支的目标发布版本来构建不同的插件，例如，在主分支中启用正在进行中的功能，而在当前发布分支中则禁用这些功能。
	static public void ConfigureGameFeaturePlugins(TargetRules Target)
	{
		// 获取日志器
		ILogger Logger = Target.Logger;
		
		//打印当前分支
		Log.TraceInformationOnce("Compiling GameFeaturePlugins in branch {0}", Target.Version.BranchName);

		//获取是否构建所有GameFeature插件
		bool bBuildAllGameFeaturePlugins = ShouldEnableAllGameFeaturePlugins(Target);

		// Load all of the game feature .uplugin descriptors
		// 加载所有的游戏特性插件描述器.
		
		//创建一个FileReference的容器
		List<FileReference> CombinedPluginList = new List<FileReference>();

		//获取所有GameFeature的插件引用
		List<DirectoryReference> GameFeaturePluginRoots = Unreal.GetExtensionDirs(Target.ProjectFile.Directory, Path.Combine("Plugins", "GameFeatures"));
		
		foreach (DirectoryReference SearchDir in GameFeaturePluginRoots)
		{
			//填充容器
			CombinedPluginList.AddRange(PluginsBase.EnumeratePlugins(SearchDir));
		}

		if (CombinedPluginList.Count > 0)
		{
			//记录所有引用到的插件,因为插件可能存在有外部依赖关系
			Dictionary<string, List<string>> AllPluginReferencesByName = new Dictionary<string, List<string>>();

			foreach (FileReference PluginFile in CombinedPluginList)
			{
				//判断找个插件是否真实存在
				if (PluginFile != null && FileReference.Exists(PluginFile))
				{
					bool bEnabled = false;
					bool bForceDisabled = false;
					
					try
					{
						//获取并添加到插件的JsonObject的字典中
						JsonObject RawObject;
						if (!AllPluginRootJsonObjectsByName.TryGetValue(PluginFile.GetFileNameWithoutExtension(), out RawObject))
						{
							RawObject = JsonObject.Read(PluginFile);
							AllPluginRootJsonObjectsByName.Add(PluginFile.GetFileNameWithoutExtension(), RawObject);
						}

						// Validate that all GameFeaturePlugins are disabled by default
						// If EnabledByDefault is true and a plugin is disabled the name will be embedded in the executable
						// If this is a problem, enable this warning and change the game feature editor plugin templates to disable EnabledByDefault for new plugins

						// 确认所有游戏功能插件默认均已禁用
						// 如果 EnabledByDefault 为真且某个插件处于禁用状态，则该插件的名称将被嵌入到可执行文件中
						// 如果出现此问题，请启用此警告，并将游戏功能编辑插件模板修改为在新插件中禁用 EnabledByDefault 参数
						
						bool bEnabledByDefault = false;
						if (!RawObject.TryGetBoolField("EnabledByDefault", out bEnabledByDefault) || bEnabledByDefault == true)
						{
							//Log.TraceWarning("GameFeaturePlugin {0}, does not set EnabledByDefault to false. This is required for built-in GameFeaturePlugins.", PluginFile.GetFileNameWithoutExtension());
						}

						// Validate that all GameFeaturePlugins are set to explicitly loaded
						// This is important because game feature plugins expect to be loaded after project startup
						
						// 确认所有游戏功能插件均已设置为“明确加载”状态
						// 这点非常重要，因为游戏功能插件需要在项目启动后才进行加载
						
						bool bExplicitlyLoaded = false;
						if (!RawObject.TryGetBoolField("ExplicitlyLoaded", out bExplicitlyLoaded) || bExplicitlyLoaded == false)
						{
							Logger.LogWarning("GameFeaturePlugin {0}, does not set ExplicitlyLoaded to true. This is required for GameFeaturePlugins.", PluginFile.GetFileNameWithoutExtension());
						}

						// You could read an additional field here that is project specific, e.g.,
						// 您在此处还可以添加一个项目特定的额外字段，例如，
						//string PluginReleaseVersion;
						//if (RawObject.TryGetStringField("MyProjectReleaseVersion", out PluginReleaseVersion))
						//{
						//		bEnabled = SomeFunctionOf(PluginReleaseVersion, CurrentReleaseVersion) || bBuildAllGameFeaturePlugins;
						//}

						if (bBuildAllGameFeaturePlugins)
						{
							// We are in a mode where we want all game feature plugins, except ones we can't load or compile
							// 我们目前处于这样一种状态：我们需要所有的游戏功能插件，但不包括那些我们无法加载或编译的插件。
							bEnabled = true;
						}

						// Prevent using editor-only feature plugins in non-editor builds
						// 防止在非编辑器版本中使用仅适用于编辑器的功能插件
						bool bEditorOnly = false;
						if (RawObject.TryGetBoolField("EditorOnly", out bEditorOnly))
						{
							if (bEditorOnly && (Target.Type != TargetType.Editor) && !bBuildAllGameFeaturePlugins)
							{
								// The plugin is editor only and we are building a non-editor target, so it is disabled
								// 该插件仅适用于编辑器使用，而我们正在构建一个非编辑器版本，因此该插件已被禁用。
								bForceDisabled = true;
							}
						}
						else
						{
							// EditorOnly is optional
							// 编辑器下专用（可选）
						}

						// some plugins should only be available in certain branches
						//  有些插件仅应在特定分支中可用
						string RestrictToBranch;
						if (RawObject.TryGetStringField("RestrictToBranch", out RestrictToBranch))
						{
							if (!Target.Version.BranchName.Equals(RestrictToBranch, StringComparison.OrdinalIgnoreCase))
							{
								// The plugin is for a specific branch, and this isn't it
								// 该插件是针对特定分支设计的，而这里并非该分支。
								bForceDisabled = true;
								Logger.LogDebug("GameFeaturePlugin {Name} was marked as restricted to other branches. Disabling.", PluginFile.GetFileNameWithoutExtension());
							}
							else
							{
								Logger.LogDebug("GameFeaturePlugin {Name} was marked as restricted to this branch. Leaving enabled.", PluginFile.GetFileNameWithoutExtension());
							}
						}

						// Plugins can be marked as NeverBuild which overrides the above
						// 可以将插件标记为“从不编译”，这将覆盖上述设置。
						bool bNeverBuild = false;
						if (RawObject.TryGetBoolField("NeverBuild", out bNeverBuild) && bNeverBuild)
						{
							// This plugin was marked to never compile, so don't
							// 此插件已被标记为永远不进行编译，所以请勿进行编译操作。
							bForceDisabled = true;
							Logger.LogDebug("GameFeaturePlugin {Name} was marked as NeverBuild, disabling.", PluginFile.GetFileNameWithoutExtension());
						}

						// Keep track of plugin references for validation later
						// 记录插件的引用信息，以便后续进行验证操作
						JsonObject[] PluginReferencesArray;
						if (RawObject.TryGetObjectArrayField("Plugins", out PluginReferencesArray))
						{
							foreach (JsonObject ReferenceObject in PluginReferencesArray)
							{
								bool bRefEnabled = false;
								if (ReferenceObject.TryGetBoolField("Enabled", out bRefEnabled) && bRefEnabled == true)
								{
									string PluginReferenceName;
									
									if (ReferenceObject.TryGetStringField("Name", out PluginReferenceName))
									{
										string ReferencerName = PluginFile.GetFileNameWithoutExtension();
										
										if (!AllPluginReferencesByName.ContainsKey(ReferencerName))
										{
											AllPluginReferencesByName[ReferencerName] = new List<string>();
										}
										AllPluginReferencesByName[ReferencerName].Add(PluginReferenceName);
									}
								}
							}
						}
					}
					catch (Exception ParseException)
					{
						Logger.LogWarning("Failed to parse GameFeaturePlugin file {Name}, disabling. Exception: {1}", PluginFile.GetFileNameWithoutExtension(), ParseException.Message);
						bForceDisabled = true;
					}

					// Disabled has priority over enabled
					// 禁用状态优先于启用状态
					if (bForceDisabled)
					{
						bEnabled = false;
					}

					// Print out the final decision for this plugin
					// 输出此插件的最终决策结果
					Logger.LogDebug("ConfigureGameFeaturePlugins() has decided to {Action} feature {Name}", bEnabled ? "enable" : (bForceDisabled ? "disable" : "ignore"), PluginFile.GetFileNameWithoutExtension());

					// Enable or disable it
					// 启用或禁用它
					if (bEnabled)
					{
						Target.EnablePlugins.Add(PluginFile.GetFileNameWithoutExtension());
					}
					else if (bForceDisabled)
					{
						Target.DisablePlugins.Add(PluginFile.GetFileNameWithoutExtension());
					}
				}
			}

			// If you use something like a release version, consider doing a reference validation to make sure
			// that plugins with sooner release versions don't depend on content with later release versions
			
			// 如果您使用的是某个发布版本，请考虑进行参考性验证，以确保那些发布版本较早的插件不会依赖于发布版本较晚的内容。
		}
	}
	
	
	
}


```
## 总结

本节构并完善了.Target.cs文件,并对插件的相关启动设置进行了验证.
此时工程可以编译成功,并顺利启动.