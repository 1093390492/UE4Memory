## UE4 多模块的插件（自定义动画节点的补充）
之前做的自定义动画节点被分成了两个模块。  
一个是动画节点功能的实现，Runtime模块，也就是MultiModifyCurveAnimNode。  
另一个就是动画的蓝图节点，Editor模块，MultiModifyCurveAnimNodeEditor，在这个模块内实现的UMultiModifyCurveAnimGraphNode以用于在蓝图内展示蓝图节点。
  
#### 创建另一个模块
我这边是直接复制的当前模块的文件夹，修改对应的文件名和各类类名。  
最终文件结构如下  
![alt 界面](https://1093390492.github.io/Image/MultModuleInPlugin/0.png)

##### 除了各个类结构的更名之外有几点要注意

1. 两个模块的Build.cs文件的修改  
	runtime模块内不能引用editor的模块
	```C##
		PublicDependencyModuleNames.AddRange(
		new string[]
		{
			"Core"
			// ... add other public dependencies that you statically link with here ...
		}
		);
		
	
	PrivateDependencyModuleNames.AddRange(
		new string[]
		{
			"CoreUObject",
			"Engine",
			// "Slate",
			// "SlateCore",
			// ... add private dependencies that you statically link with here ...	
		}
		);
	```
	editor模块可以引入所需要的其他editor模块。
	```C#
	PublicDependencyModuleNames.AddRange(
	new string[]
	{
		"Core",
		"AnimGraph",
		"BlueprintGraph",
		"MultiModifyCurveAnimNode"
		// ... add other public dependencies that you statically link with here ...
	}
	);
	

	PrivateDependencyModuleNames.AddRange(
	new string[]
	{
		"CoreUObject",
		"Engine"//,
		// "Slate",
		// "SlateCore",
		// ... add private dependencies that you statically link with here ...	
	}
	);
	```
2.  修改复制出来的IMPLEMENT_MODULE宏
	```C++
	// MultiModifyCurveAnimGraphNodeModule.cpp
	IMPLEMENT_MODULE(FMultiModifyCurveAnimGraphNodeModule, MultiModifyCurveAnimNodeEditor)
	```

3. 修改插件的uplugin
	```
	{
	"FileVersion": 3,
	"Version": 1,
	"VersionName": "1.0",
	"FriendlyName": "MultiModifyCurveAnimNode",
	"Description": "支持使用TMap<TName,float>做变形动画，内置映射表。",
	"Category": "Other",
	"CreatedBy": "Meteoc",
	"CreatedByURL": "",
	"DocsURL": "",
	"MarketplaceURL": "",
	"SupportURL": "",
	"CanContainContent": true,
	"IsBetaVersion": false,
	"IsExperimentalVersion": false,
	"Installed": false,
	"Modules": [
		{
			"Name": "MultiModifyCurveAnimNode",
			"Type": "Runtime",
			"LoadingPhase": "PreLoadingScreen"
		},
		{
			"Name": "MultiModifyCurveAnimNodeEditor",
			"Type": "UncookedOnly",
			"LoadingPhase": "PreLoadingScreen"
		}
	]
	}
	```
	这个Name ： MultiModifyCurveAnimNodeEditor就是上面IMPLEMENT_MODULE的第二个参数。
	重点在于这个"Type": "UncookedOnly"。Type如果为Editor，编辑器会警告，如果是runtime则会在打包时报错。UncookedOnly则在Cook时跳过这个模块。
	以下是各类Type含义
	```C# 
	* Environment that can load a module.
	*/
	namespace EHostType
	{
		enum Type
		{
		// Loads on all targets, except programs.
		Runtime,	//Runtime模式下会去加载
		
		// Loads on all targets, except programs and the editor running commandlets.
		RuntimeNoCommandlet,	//Runtime模式, 不包含命令
		
		// Loads on all targets, including supported programs.
		RuntimeAndProgram,		//Runtime模式, 小程序
		
		// Loads only in cooked games.
		CookedOnly,		//和Cook资源相关

		// Only loads in uncooked games.
		UncookedOnly, //只在没有cook过的程序中加载

		// Deprecated due to ambiguities. Only loads in editor and program targets, but loads in any editor mode (eg. -game, -server).
		// Use UncookedOnly for the same behavior (eg. for editor blueprint nodes needed in uncooked games), or DeveloperTool for modules
		// that can also be loaded in cooked games but should not be shipped (eg. debugging utilities).
		Developer,	//开发模式下加载,兼容老版本留下来的枚举

		// Loads on any targets where bBuildDeveloperTools is enabled.
		DeveloperTool,	//开发模式下加载 新的枚举

		// Loads only when the editor is starting up.
		Editor,		//Editor模块
		
		// Loads only when the editor is starting up, but not in commandlet mode.
		EditorNoCommandlet,	//Editor非命令行加载

		// Loads only on editor and program targets
		EditorAndProgram,	//Editor小程序

		// Only loads on program targets.
		Program,	//独立程序
		
		// Loads on all targets except dedicated clients.
		ServerOnly,		//服务端模块
		
		// Loads on all targets except dedicated servers.
		ClientOnly,		//客户端模块

		// Loads in editor and client but not in commandlets.
		ClientOnlyNoCommandlet,		//客户端没有命令行的模块
		
		//~ NOTE: If you add a new value, make sure to update the ToString() method below!
		Max	//如果新加枚举的话 别忘了把这个Namespace中的tostring方法完善一下
	};

	```