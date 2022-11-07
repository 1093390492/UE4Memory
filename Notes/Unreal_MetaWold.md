## UE4 不常用却有用宏列表

#### UPARAM
用于声明参数为引用传递
```
UPARAM(ref) //C++函数在蓝图内的引用传参
UPARAM(DisplayName = "早上好（啦啦啦）") //自定义参数的显示名称 支持中文和特殊字符
```

``` C++
	UFUNCTION(BlueprintCallable)
	static void RefFuncTest(UPARAM(ref)int& RefValue){}

	UFUNCTION(BlueprintCallable)
	static void CustomParamDisplayTest(UPARAM(DisplayName = "早上好（啦啦啦）")int& RefValue){}

```
![alt 蓝图节点效果](https://1093390492.github.io/Image/Unreal_MetaWold/0.png)

---


#### CustomThunk
一般用于声明泛型蓝图函数 需要配合CustomStructureParam
```
CustomThunk UnrealHeaderTool //代码生成器将不为此函数生成thunk，用户需要自己通过 DECLARE_FUNCTION 或 DEFINE_FUNCTION 宏来提供thunk。
```

#### WorldContext
是WorldContext自动获取调用它的对象,将其赋值给它指定的参数meta=(WorldContext="WorldContextObject")
```C++
	UFUNCTION(BlueprintCallable,meta = (WorldContext = "WorldContent"))
	static UBussinesManagerSubsystem* GetBussinesManagerInstance(UObject* WorldContent);
```

#### EditInlineNew
表示此类的对象可以从虚幻编辑器属性窗口创建，而不是从现有资源中引用。  默认行为是只能通过“属性”窗口分配对现有对象的引用）。  此说明符传播到所有子类；  子类可以使用 NotEditInlineNew 说明符覆盖它
``` C++
UCLASS(EditInlineNew)
class EMPTYPROJECT_API UMyObject : public UObject
{
	GENERATED_BODY()
public:
	UPROPERTY(EditAnywhere,BlueprintReadWrite)
	float Life;
};
```
#### Instanced
配合EditInlineNew使用 标记对象为独立创建的类型 
``` C++
	UPROPERTY(Instanced,EditAnywhere,BlueprintReadWrite)
	UMyObject* MyClass;
```

#### InlineEditConditionToggle && EditCondition && Units
InlineEditConditionToggle 表示出布尔属性只内联显示为其他属性中的一个编辑条件切换，不应显示在其自身的行上。
人话就是这个可以用来表示另一个属性编辑的激活与未激活
``` C++
UPROPERTY(EditAnywhere, meta=(InlineEditConditionToggle))
bool bCanFly;

UPROPERTY(EditAnywhere, meta=(EditCondition="bCanFly", Units="s"))
float FlapPeriodSeconds;
```
![alt 蓝图节点效果](https://1093390492.github.io/Image/Unreal_MetaWold/1.gif)

#### MustImplement=""
用于 Subclass 和 SoftClass 属性。指示所选类必须实现特定接口
UPROPERTY(EditAnywhere, meta=(MustImplement="InteractibleInterface"))
TSubclassOf<AActor> InteractibleActor;