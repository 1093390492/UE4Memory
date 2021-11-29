## 蓝图函数异步节点
在蓝图中类似delay节点亦或DownloadImage节点，会等待某个条件或是任务完成才会执行某个exec node  
这里以延迟一帧为例  
继承UBlueprintAsyncActionBase
```C++
#include "CoreMinimal.h"
#include "Kismet/BlueprintAsyncActionBase.h"
#include "DelayOneFrame.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FDelayOneFrameOutputPin,float,OneParam);

/**
 * 
 */
UCLASS()
class MULTITHREADPROJECT_API UDelayOneFrame : public UBlueprintAsyncActionBase
{
	GENERATED_BODY()
	
public:

    ///异步执行的节点 想要多节点可以多塞几个成员变量
	UPROPERTY(BlueprintAssignable)
	FDelayOneFrameOutputPin AfterOneFrame;

    ///开放给蓝图的函数节点
	UFUNCTION(BlueprintCallable,meta = (BlueprintInternalUseOnly = "true",WorldContext = "WorldContextObject"),Category = "Flow Control")
	static UDelayOneFrame* WaitForOneFrame(const UObject* WorldContextObject,const float SomeInputVariables);

    ///该节点激活时执行函数 这里用于使用计时器等待一帧的时间
	virtual  void Activate() override;

    
	void ExecuteAfterOneFrame();
	
private:
	const UObject* WorldContextObject = nullptr;

    /// 函数入参 这里没有实际使用到
	float MyFloatInput = 0.0f;
	
};
```

```C++
#include "DelayOneFrame.h"

UDelayOneFrame* UDelayOneFrame::WaitForOneFrame(const UObject* WorldContextObject, const float SomeInputVariables)
{

	UDelayOneFrame* BlueprintNode = NewObject<UDelayOneFrame>();
	BlueprintNode->WorldContextObject = WorldContextObject;

	BlueprintNode->MyFloatInput = SomeInputVariables;

	return  BlueprintNode;
}

void UDelayOneFrame::Activate()
{
	Super::Activate();

	WorldContextObject->GetWorld()->GetTimerManager().SetTimerForNextTick(this,&UDelayOneFrame::ExecuteAfterOneFrame);

	
}

void UDelayOneFrame::ExecuteAfterOneFrame()
{
	AfterOneFrame.Broadcast(MyFloatInput);
}

```
