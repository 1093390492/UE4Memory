## UE4 自定义命令行变量
这里说的UE4中配置文件（Config）指的是.ini文件。
我们所见到的配置文件一般只存在于以下四个路径。

1. \Engine\Config
2. \Engine\Saved\Config
3. Projects\[ProjectName]\Config 在编辑器内对项目的配置
4. Projects\[ProjectName]\Saved\Config 游戏项目运行后生成 记录运行时修改的配置

#### 一些规则
  
|  |  |
| - | - |
| UCLASS(defaultconfig) |  defaultconfig:配置会写入到引擎默认配置里, 路径一般是./Config 另外打包出来后运行以后会自动生成配置文件信息,但是无法再手动修改ini文件; |
| UCLASS(configdonotcheckdefaults) | configdonotcheckdefaults可以运行时修改但是会保存到./Save/Config/Windows内对应名称的ini内 |
| UCLASS(config=FileName) | 表示这个类默认条件下将配置信息保存到哪个配置文件，config后面的文件名可以是任意字符串。 |
| UCLASS(perObjectConfig) | 表示这个类的配置信息将会基于每个实例进行存储。 |
| UCLASS(config=XXX,configdonotcheckdefaults) | 表示这个类对应的配置文件不会检查XXX层级上层的DefaultXXX配置文件是否有该信息（后面会解释层级），就直接存储到Saved目录下。 |
| UPROPERTY(config) | 不指定Section的情况下，标记config的这个属性在保存到配置文件里面的时候会保存在当前类对应的Section部分。同理，加载的时候也会从当前类对应的Section下加载。 |
| UPROPERTY(transient) | 隐藏当前属性 即不保存到配置文件中 |
| UPROPERTY(globalconfig) | 不指定Section的情况下，标记config的这个属性在保存到配置文件里面的时候会保存在基类对应的Section部分。同理，加载的时候也会从基类对应的Section下加载。 |


ini结构
```
[/Script/Engine.Console]   //标题即Section
HistoryBuffer=TestLoadPak   //key = value

[/Script/Engine.Console_2]   //Section_2
HistoryBuffer2=TestLoadPak2   //key = value
```
————————————————  

### 继承UDeveloperSettings
继承UDeveloperSettings的类，虚幻会自动的完成保存读取的工作，并且会在setting菜单中显示，
————————————————
```C++
//注释可以显示在编辑器Setting标题之下
UCLASS(Config = CustomSetting ,defaultconfig/*configdonotcheckdefaults*/)
class PLUGINSLEARN_API UCustomDeveloperSettings : public UDeveloperSettings
{
	GENERATED_BODY()

public:

	//决定是出现在Editor Preferences里还是Project Setting里 可选项[Editor] [Project]
	virtual FName GetContainerName() const override {return TEXT("Project");}

	//setting菜单里item名称
	virtual FName GetCategoryName() const override {return  TEXT("CustomSettings");}

	// 唯一名称 一般用类名
	virtual FName GetSectionName() const override {return TEXT("CustomSettings");}


	// 获取CDO 借此获取配置数据
	UFUNCTION(BlueprintPure,DisplayName = "CustomSetting")
	static UCustomDeveloperSettings* Get(){return  GetMutableDefault<UCustomDeveloperSettings>();}

    //UPROPERTY需要Config标记
	UPROPERTY(Config,EditAnywhere,BlueprintReadWrite,Category = "Category_0")
	float FloatSetting = 10;
	
	UPROPERTY(Config,EditAnywhere,BlueprintReadWrite,Category = "Category_0")
	FString StringSetting = "Hello";
};
```
编译完成之后可以看到如下效果  
![alt Setting界面](https://1093390492.github.io/Image/UE4Config/0.png)  
如果在setting界面修改数值，则根据UCLASS是defaultconfig还是configdonotcheckdefaults在 Projects\[ProjectName]\Config 或 Projects\[ProjectName]\Saved\Config下找到DefaultCustomSetting.ini被修改后的结果  

---
接下来就可以直接在蓝图中取到对应的config数据  
![alt Blueprint界面](https://1093390492.github.io/Image/UE4Config/1.png)  

----

### 对一个UObject进行配置

让一个UObject对象绑定Config需要使用到UObject::LoadConfig和UObject::SaveConfig函数

```C++
UCLASS(BlueprintType,Config = ObjectConifg)
class PLUGINSLEARN_API UObjectSetting : public UObject
{
	GENERATED_BODY()
public:
	UPROPERTY(Config,EditAnywhere,BlueprintReadWrite,Category = OneCategory)
	int Voice = 10;

	UPROPERTY(Transient,EditAnywhere,BlueprintReadWrite,Category = OneCategory)
	int Speed = 10;
public:
	UFUNCTION(BlueprintCallable)
	void LoadSetting()
	{
		LoadConfig();
	}
	UFUNCTION(BlueprintCallable)
	void SaveSetting()
	{
		SaveConfig();
	}
	
};
```
在蓝图中调用  
![alt Blueprint界面](https://1093390492.github.io/Image/UE4Config/2.png) 



参考资料  
[UE4添加自定义项目设置 ](http://supervj.top/2020/08/31/%E6%89%A9%E5%B1%95%E6%B8%B8%E6%88%8F%E8%AE%BE%E7%BD%AE/)  
[UE4 Config配置文件详解](https://blog.csdn.net/u012999985/article/details/52801264)