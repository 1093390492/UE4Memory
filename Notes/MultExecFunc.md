## 用户自定蓝图函数多执行引脚
核心代码  
UFUNCTION(BlueprintCallable, **meta = (ExpandEnusmAsExecs = "Branches")** )  
Branches为一个UENUM枚举，结构成员即为引脚列表
```C++
UENUM(BlueprintType)
enum class EMyEnum :uint8
{
	OneExec,
	TwoExec
};

UFUNCTION(BlueprintCallable,meta = (ExpandEnusmAsExecs = "Branches"))
static void MultExec(bool BrancheSwitch,EMyEnum& Branches);
```

```C++
void UUtiliyFunctionLibrary::MultExec(bool BrancheSwitch, EMyEnum& Branches)
{
	Branches = BrancheSwitch?EMyEnum::OneExec:EMyEnum::TwoExec;
}

```
 引用参数Branches的输出值决定执行哪一个引脚