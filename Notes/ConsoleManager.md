## UE4 自定义命令行
### 在命令行调用UFUCTION

以下类型可以直接声明UFUNCTION成员函数为Exec函数
* Possessed Pawns 
* Player Controllers
* Player Input
* Cheat Managers
* Game Modes
* Game Instances 
* overriden Game 
* Engine classes  

声明一个无参Exec函数  
```C++
   UFUNCTION(Exec)
   void YourExecFunction();
```  
![alt Console调用](https://1093390492.github.io/Image/ConsoleManager/0.png) 

带参数Exec
```C++
   UFUNCTION(Exec)
   void YourExecFunction(int32 arg1, FString arg2);
```  
![alt Console调用](https://1093390492.github.io/Image/ConsoleManager/1.png)   

以上参考[UE GuideBook Exec Functions](https://unreal.gg-labs.com/wiki-archives/common-pitfalls/exec-functions)  

#### 自定义命令行变量

使用命令行控制变量
```C++
// 声明一个控制台bool类型变量 CVarCustomVoiceEnable，cmd命令为custom.voiceEnable，默认值为True，提示为Custom Set Open Voice，
//flag使用默认ECVF_Default flag的含义可以查看IConsoleManager.h
static TAutoConsoleVariable<bool> CVarCustomVoiceEnable(TEXT("custom.voiceEnable"),true,TEXT("Custom Set Open Voice"),ECVF_Default);
```
![alt Console调用](https://1093390492.github.io/Image/ConsoleManager/2.png)   

---
```C++
//声明string类型变量
static TAutoConsoleVariable<FString> CVarCustomName(TEXT("custom.name"),"Tom",TEXT("Custom Name"),ECVF_Default);

```
![alt Console调用](https://1093390492.github.io/Image/ConsoleManager/3.png)   

---

获取命令行变量数值
```C++
bool VoiceEnable = CVarCustomVoiceEnable.GetValueOnGameThread();
FString CustomName = CVarCustomName.GetValueOnGameThread();
```

#### 追踪cmd变量的变化
##### 1. Tick轮询（推荐）
在一个Tick函数内轮询检查变量的状态
```C++ 
//.h
#pragma once

#include "CoreMinimal.h"
#include "Tickable.h"

static TAutoConsoleVariable<bool> CVarCustomVoiceEnable(TEXT("custom.voiceEnable"),true,TEXT("Custom Set Open Voice"),ECVF_Default);

static TAutoConsoleVariable<FString> CVarCustomName(TEXT("custom.name"),"Tom",TEXT("Custom Name"),ECVF_Default);

class  ConsoleVarManager  : public FTickableGameObject
{
public:
	//
	bool VoiceEnable = CVarCustomVoiceEnable.GetValueOnGameThread();
	FString CustomName = CVarCustomName.GetValueOnGameThread();
private:
	virtual void Tick(float DeltaTime) override
	{
		if(VoiceEnable != CVarCustomVoiceEnable.GetValueOnGameThread())
		{
			VoiceEnable = !VoiceEnable;
			GEngine->AddOnScreenDebugMessage(INDEX_NONE,3,FColor::Red, FString::Printf(TEXT("VoiceEnable : %s"), (VoiceEnable?TEXT("True"):TEXT("False"))));
		}
		
		if(CustomName != CVarCustomName.GetValueOnGameThread())
		{
			CustomName = CVarCustomName.GetValueOnGameThread();
			GEngine->AddOnScreenDebugMessage(INDEX_NONE,3,FColor::Red,"CustomName : " + CustomName);
		}
		
	}
public:
// 纯C++函数继承FTickableGameObject需要实现GetStatId和Tick函数
virtual TStatId GetStatId() const override
{
	RETURN_QUICK_DECLARE_CYCLE_STAT(ConsoleVarManager, STATGROUP_Tickables);
}
```
提示：因为这里ConsoleVarManager没有继承UObject所以需要构造之后才能触发Tick函数，如果继承UObject会随着ODC的创建而触发Tick

##### 2. 注册AutoConsoleVariableSink
```C++
//.cpp
FAutoConsoleVariableSink CMyVarSink(FConsoleCommandDelegate::CreateLambda(
[]()
{
	//在这个函数内手动检查各个变量的状态

	UE_LOG(LogTemp,Log,TEXT("Sink CustomName Change %s"), * (CVarCustomName.GetValueOnGameThread()));
}
));
```

##### 3. 注册回调函数（不推荐）

```C++

	ConsoleVarManager():FTickableGameObject()
	{
		CVarCustomVoiceEnable.AsVariable()->SetOnChangedCallback(
			FConsoleVariableDelegate::CreateStatic(	[](IConsoleVariable* MyVar)
			{
				//do something
				UE_LOG(LogTemp,Log,TEXT("Voice Change %s"),MyVar->GetBool()?TEXT("True"):TEXT("False"));
			})
		);
	}

```
文档标注  
`
最后一种方法是使用回调，建议尽量不使用。如不谨慎使用，可能会引起问题：
循环可能出现死锁（可以防止死锁出现，但使用哪个回调则不明确）。
回调可在 !Set() 被调用中的任意时间处返回。代码必须在所有情况下（在初始化中、在序列化中）均能使用。 可假定其固定处于主线程一侧。 
不建议使用此方法，除非用上述其他方法已无法解决。 
`
   