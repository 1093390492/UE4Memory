## 非AActor类的Tick实现

1. 继承虚基类FTickableGameObject
``` C++
#include "Tickable.h"
class  ConsoleVarManager  : public FTickableGameObject
{
	//...
}
```

2. 实现虚函数Tick  
``` C++
virtual void Tick(float DeltaTime) override
{
	//do something in Tick		
}
```
3. 实现GetStatId，如果该类继承了UObject，则在Uobject里已实现GetStatId
```C++
virtual TStatId GetStatId() const override
{
	RETURN_QUICK_DECLARE_CYCLE_STAT(ConsoleVarManager, STATGROUP_Tickables);
}
```

在FTickableGameObject的构造函数内实现了注册该类至TickableObjects
```C++
FTickableGameObject::FTickableGameObject()
{
	FTickableStatics& Statics = FTickableStatics::Get();

	if (UObjectInitialized())
	{
		Statics.QueueTickableObjectForAdd(this);
	}
	else
	{
		AddTickableObject(Statics.TickableObjects, this);
	}
}
```
因此可知继承了FTickableGameObject的类需要首先执行父类的构造函数才能被注册调用Tick的方法。
这里举例挂载到了GameInstance内
```C++
#pragma once

#include "CoreMinimal.h"
#include "Engine/GameInstance.h"
#include "ConsoleVarManager.h"
#include "MyGameInstance.generated.h"

/**
 * 
 */
UCLASS()
class PLUGINSLEARN_API UMyGameInstance : public UGameInstance
{
	GENERATED_BODY()

	ConsoleVarManager* ConsoleManager;

	virtual void Init() override
	{
		ConsoleManager = new ConsoleVarManager();
		UE_LOG(LogTemp,Warning,TEXT("UMyGameInstance Init"));
		Super::Init();
	}

	virtual void Shutdown() override
	{

		delete ConsoleManager;
		ConsoleManager = nullptr;
		UE_LOG(LogTemp,Warning,TEXT("UMyGameInstance Shutdown"));
		Super::Shutdown();
	}
};
```
或者直接多继承UObject，在引擎构造该类的ODC即调用了父类的构造函数。