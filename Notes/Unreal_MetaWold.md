## UE4 不常用却有用Meta宏列表

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

