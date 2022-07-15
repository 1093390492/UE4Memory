##UE5扩展编辑器菜单
####创建一个工具栏按钮插件来启动UMG（设定UI Style）

资源目录如下
![alt 界面](https://1093390492.github.io/Image/EditorMenu/2.png)

打开XXXStyle.cpp
``` C++
const FVector2D Icon16x16(16.0f, 16.0f);
const FVector2D Icon20x20(20.0f, 20.0f);
const FVector2D Icon128x128(128.0f, 128.0f); //声明分辨率
// const FVector2D Icon256x256(256.0f, 256.0f);
TSharedRef< FSlateStyleSet > FObjectColorMarkStyle::Create()
{
	TSharedRef< FSlateStyleSet > Style = MakeShareable(new FSlateStyleSet("ObjectColorMarkStyle"));

	//设定搜索Icon资源的根目录
	Style->SetContentRoot(IPluginManager::Get().FindPlugin("ObjectColorMark")->GetBaseDir() / TEXT("Resources"));

	//Style->Set("ObjectColorMark.PluginAction", new IMAGE_BRUSH_SVG(TEXT("PlaceholderButtonIcon"), Icon20x20));

	/** Style->Set为这个style增加一个FSlateBrush
	* “"ObjectColorMark.PluginAction"”参数为标定的Style下的FSlateBrush的名称  后面
	* new IMAGE_BRUSH (TEXT("Icon128"), Icon128x128) 
	* TEXT("Icon128") 上面设置的资源的根目录下为不带后缀的图片资源文件名 Icon128x128为分辨率
	*/
	Style->Set("ObjectColorMark.PluginAction", new IMAGE_BRUSH (TEXT("Icon128"), Icon128x128));

	//如果只有一个 在不设定Ui元素的icon时候 会使用这个默认的

	Style->Set("ObjectColorMark.LALA", new IMAGE_BRUSH_SVG (TEXT("PlaceholderButtonIcon"), Icon20x20));
	return Style;
}
```
这里设置的FSlateBrush可以在FSlateIcon中使用
``` C++
FSlateIcon(FObjectColorMarkStyle::GetStyleSetName(), "ObjectColorMark.PluginAction")
```

---------------
``` C++
void FObjectColorMarkModule::StartupModule()
{
	// This code will execute after your module is loaded into memory; the exact timing is specified in the .uplugin file per-module
	
	FObjectColorMarkStyle::Initialize(); //初始化style 内部会执行 上面的 Create函数
	FObjectColorMarkStyle::ReloadTextures(); // 按照Create设定的读取目录加载硬盘图片资源

	FObjectColorMarkCommands::Register(); //构造单例并且注册Command
	
	PluginCommands = MakeShareable(new FUICommandList);


	// 映射Commad的回调函数 FCanExecuteAction()参数位为判断执行条件函数回调
	PluginCommands->MapAction(
		FObjectColorMarkCommands::Get().PluginAction,
		FExecuteAction::CreateRaw(this, &FObjectColorMarkModule::PluginButtonClicked),
		FCanExecuteAction());

	// 注册ToolMenus上的启动回调 在ToolMenus启动执行RegisterMenus 
	UToolMenus::RegisterStartupCallback(FSimpleMulticastDelegate::FDelegate::CreateRaw(this, &FObjectColorMarkModule::RegisterMenus));


}
```

``` C++
void FObjectColorMarkModule::RegisterMenus()
{
	// Owner will be used for cleanup in call to UToolMenus::UnregisterOwner
	FToolMenuOwnerScoped OwnerScoped(this);	// FToolMenuOwnerScoped构造的时候会把this传入OwnerStack
	{
		// "LevelEditor.MainMenu.Window"名称是Menu创建的时候注册的  UE5 ui调整 代码有所变化 逻辑依然类似
		/*也有类似这种写法 UToolMenu* PlayToolBar = UToolMenus::Get()->RegisterMenu("LevelEditor.LevelEditorToolBar.PlayToolBar", NAME_None, EMultiBoxType::SlimHorizontalToolBar);

		/*
		FLevelEditorMenu::RegisterLevelEditorMenus

		void FLevelEditorMenu::RegisterLevelEditorMenus()
		{
			struct Local
			{
					//...
			};

			UToolMenus* ToolMenus = UToolMenus::Get();
			ToolMenus->RegisterMenu("LevelEditor.MainMenu", "MainFrame.MainMenu", EMultiBoxType::MenuBar);
			ToolMenus->RegisterMenu("LevelEditor.MainMenu.File", "MainFrame.MainTabMenu.File");
			ToolMenus->RegisterMenu("LevelEditor.MainMenu.Window", "MainFrame.MainMenu.Window");

			// Add level loading and saving menu items
			Local::RegisterFileLoadAndSaveItems();

			// Add recent / favorites
			Local::FillFileRecentAndFavoriteFileItems();

			// Extend the Edit menu
			Local::ExtendEditMenu();

			// Extend the Help menu
			Local::ExtendHelpMenu();
		}
		*/

		UToolMenu* Menu = UToolMenus::Get()->ExtendMenu("LevelEditor.MainMenu.Window");
		{
			FToolMenuSection& Section = Menu->FindOrAddSection("WindowLayout");
			Section.AddMenuEntryWithCommandList(FObjectColorMarkCommands::Get().PluginAction, PluginCommands);
		}
	}

	{
		UToolMenu* ToolbarMenu = UToolMenus::Get()->ExtendMenu("LevelEditor.LevelEditorToolBar.PlayToolBar");
		{
			FToolMenuSection& Section = ToolbarMenu->FindOrAddSection("Play"); //
			{
				FToolMenuEntry& Entry = Section.AddEntry(FToolMenuEntry::InitToolBarButton(FObjectColorMarkCommands::Get().PluginAction));
				Entry.SetCommandList(PluginCommands);
				
				// 想要自定义或者有多个Icon设定可以单独设定Entry的Icon属性
				Entry.Icon = FSlateIcon(FObjectColorMarkStyle::GetStyleSetName(),"ObjectColorMark.LALA");

			}
		}
	}
}
```
特别值得一提的是ToolbarMenu->FindOrAddSection("XXX") 获取的Section得首先确定UToolMenus::Get()->ExtendMenu。
例如像放到Play之后
![alt 界面](https://1093390492.github.io/Image/EditorMenu/0.png)
就得获取"LevelEditor.LevelEditorToolBar.PlayToolBar"，
如果要放到Content之后就需要获取"LevelEditor.LevelEditorToolBar.AssetsToolBar"并且ToolbarMenu->FindOrAddSection("Play")。
![alt 界面](https://1093390492.github.io/Image/EditorMenu/1.png)
具体Name得翻源码寻找。
在实际开发中自己扩展出来的自定义Menu即可

最后是启动目录下的Editor UMG
```C++
void FObjectColorMarkModule::GenerateWidget()
{

	UBlueprint* WidgetBlueprint = LoadObject<UEditorUtilityWidgetBlueprint>(nullptr,TEXT("EditorUtilityWidgetBlueprint'/ObjectColorMark/MenuToolList.MenuToolList'"));

	if(!WidgetBlueprint)
	{
		UE_LOG(LogTemp,Error,TEXT("EditorUtilityWidgetBlueprint'/ObjectColorMark/MenuToolList.MenuToolList' No Found"));
		return;
	}
	if(WidgetBlueprint)
	{
		if (WidgetBlueprint->GeneratedClass->IsChildOf(UEditorUtilityWidget::StaticClass()))
		{
			UEditorUtilityWidgetBlueprint* EditorUtilityWidget = Cast<UEditorUtilityWidgetBlueprint>(WidgetBlueprint);
			if(EditorUtilityWidget)
			{
				UEditorUtilitySubsystem* EditorUtilitySubsystem = GEditor->GetEditorSubsystem<UEditorUtilitySubsystem>();
				EditorUtilitySubsystem->SpawnAndRegisterTab(EditorUtilityWidget);
			}
		}
	}
	
}
```