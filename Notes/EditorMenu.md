## 使用UMG制作插件

以往UE4制作带UI界面的插件需要使用Slate来编写UI，虽然说Slate为适配UI做了大量的宏，但说到底纯代码构筑UI还是反直觉。  
UE4.26版本后添加了EditorUilityWidget类型，可以使用UMG制作UI，并且直接在编辑状态直接执行。  
右键创建资源Editor Utilities ->Editor Utility Widget
右键Editor Utility Widget资源Run Editor Utility Widget即可直接在编辑器状态运行该Widget。
## 例
首先设计一个文本输入框的UI界面，一个多行输入和一个按钮。  
![alt 界面](https://1093390492.github.io/Image/UMG2Slate/0.png)

制作按钮点击的蓝图事件。  
![alt 点击事件](https://1093390492.github.io/Image/UMG2Slate/1.png)

在编辑器中运行这个Editor Utility Widget，效果如下  
![alt 效果](https://1093390492.github.io/Image/UMG2Slate/2.png)  

---

然而在实际开发使用中，一般的插件要是都用右击运行Editor Utility Widget的方式还是繁琐，尤其是制作的多了不便于管理，所以还是需要结合Slate来制作插件。
## 例
1. 首先在UE4内创建一个插件，选择独立窗口的模板 Editor Standalone Window,起名 OpenTextEditor，创建完成后记得修改插件的偏好，勾上Can Contain Content，这样才能让这个插件附带资源，我们需要创建Editor Utility Widget给插件使用。 
2. 将我们刚刚创建的Editor Utility Widget移动到 Editor Utility Widget的Content资源文件夹内。
3. 创建完成后编辑器内是看不到插件的代码的，因为UE4帮我们创建的类并不继承UClass，需要到IDE内打开OpenTextEditorModule编辑。


---
核心代码
头文件添加一个 TSharedPtr\<SWidget\> EditorBoxWidget 用于保存从uusewidget拿到的slate类widget. 
GenerateWidget函数用于从本地Editor Utility Widget生成我们所需要的SWidget。
```C++
#pragma once

#include "CoreMinimal.h"
#include "Modules/ModuleManager.h"

class FToolBarBuilder;
class FMenuBuilder;

class FOpenTextEditorModule : public IModuleInterface
{
public:

	/** IModuleInterface implementation */
	virtual void StartupModule() override;
	virtual void ShutdownModule() override;
	
	/** This function will be bound to Command (by default it will bring up plugin window) */
	void PluginButtonClicked();
	
private:

	void RegisterMenus();

	TSharedRef<class SDockTab> OnSpawnPluginTab(const class FSpawnTabArgs& SpawnTabArgs);
	
	void GenerateWidget();

    ///保存从uusewidget拿到的slate类widget
	TSharedPtr<SWidget> EditorBoxWidget;

private:
	TSharedPtr<class FUICommandList> PluginCommands;
};
```
cpp文件
```C++

TSharedRef<SDockTab> FOpenTextEditorModule::OnSpawnPluginTab(const FSpawnTabArgs& SpawnTabArgs)
{
///实际上GenerateWidget放到StartupModule函数内执行较好，这里为了文档结构，放到该函数内执行。
	GenerateWidget();

	if(!EditorBoxWidget.IsValid())
	{
		return SNew(SDockTab)
		.TabRole(ETabRole::NomadTab)
		[
			// Put your tab content here!
			SNew(SBox)
			.HAlign(HAlign_Center)
			.VAlign(VAlign_Center)
			[
				SNew(STextBlock)
				.Text(LOCTEXT("WindowWidgetText", "EditorWidget is NULL"))
			]
		];
	}
	else
	{
		return SNew(SDockTab)
		.TabRole(ETabRole::NomadTab)
		[
		EditorBoxWidget.ToSharedRef()
		];
	}

}

...

void FOpenTextEditorModule::GenerateWidget()
{
	UUserWidget* UserWidget = nullptr;
	
    ///EditorUtilityWidgetBlueprint'/OpenTextEditor/TextEditorUMG.TextEditorUMG为我Editor Utility Widget的引用路径 后面加_C表示读取Class文件。
	UClass* loadedClass = LoadClass<UUserWidget>(nullptr,TEXT("EditorUtilityWidgetBlueprint'/OpenTextEditor/TextEditorUMG.TextEditorUMG_C'"));

	if(!loadedClass)
	{
		UE_LOG(LogTemp,Error,TEXT("EditorUtilityWidgetBlueprint'/OpenTextEditor/TextEditorUMG.TextEditorUMG_C' No Found"));
		return;
	}

	UUserWidget* widget = CreateWidget<UUserWidget>(GEditor->GetEditorWorldContext().World(),loadedClass);
	
	EditorBoxWidget = widget->TakeWidget();
}

```
编译->重启Ue4->点击工具栏的OpenTExtEditor->正常打开制作的UMG界面