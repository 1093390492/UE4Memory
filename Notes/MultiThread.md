
## 多线程的几种方案
以计算素数为例

---
最常用也是最简单的
### AsyncTask

``` C++
AsyncTask(ENamedThreads::GameThread, [mainMessage, byteDataArray, clientConnectionIDGlobal, tcpClientGlobal, socketClientGlobal]() {
	if (socketClientGlobal != nullptr)
		socketClientGlobal->onreceiveTCPMessageEventDelegate.Broadcast(mainMessage, byteDataArray, clientConnectionIDGlobal);
	if (tcpClientGlobal != nullptr)
		tcpClientGlobal->onreceiveTCPMessageEventDelegate.Broadcast(mainMessage, byteDataArray, clientConnectionIDGlobal);
});
```
以一个TCP通讯代码节选为例
第一个参数ENamedThreads::GameThread设定任务跑在那个线程上，值得一提的是，即使设定为ENamedThreads::GameThread这个地方也不会阻塞，由UE决定在某个时间在游戏线程执行任务。

---
### Async
``` C++
void UUtiliyFunctionLibrary::CalculatePrime(int MaxNum)
{
	
	TFuture<TArray<int>> Result1 = Async(EAsyncExecution::ThreadPool,[=]()
	{
		for(int i = 1;i<MaxNum;i++)
		{
			bool isPrime = true;
	
			for(int j = 2; j <= i / 2; ++j)
			{
				if(i % j == 0)
				{
					isPrime = false;
					break;
				}
			}

			if(isPrime)
			{
				UE_LOG(LogTemp,Log,TEXT("Donw %d"),i);
			}
		}
		TArray<int> temp;
		return temp;
	});
}
```
使用模板函数Async 返回一个TFuture的的模板类，TFuture包含当前异步处理的状态信息，可使用timer或者在tick中轮询。

---

### Task
``` C++
#pragma once

#include "CoreMinimal.h"
#include "Async/AsyncWork.h"
class MULTITHREADPROJECT_API FCalculateTask : public FNonAbandonableTask
{
public:
	FCalculateTask(int Max):ToCalculateMax(Max){}

    //析构函数可省略 或者进行资源回收
	~FCalculateTask()
	{
		UE_LOG(LogTemp,Warning,TEXT("UCalculateTask End"));
	}
	
    ///固定写法
	FORCEINLINE TStatId GetStatId() const
	{
		RETURN_QUICK_DECLARE_CYCLE_STAT(UCalculateTask,STATGROUP_ThreadPoolAsyncTasks);
	}

	int ToCalculateMax;
	
	TArray<int> PrimeList;
	
    /// 实际的多线程工作
	void DoWork();
	
private:
	bool IsPrime(int value);

};
```

``` C++
#include "CalculateTask.h"
void FCalculateTask::DoWork()
{
	PrimeList.Empty();
	for(int i = 1;i<ToCalculateMax;i++)
	{
		if(IsPrime(i))
		{
			PrimeList.Add(i);
			UE_LOG(LogTemp,Log,TEXT("Done %d"),i);
		}
	}
}

bool FCalculateTask::IsPrime(int value)
{
	bool isPrime = true;
	for(int i = 2; i <= value / 2; ++i)
	{
		if(value % i == 0)
		{
			isPrime = false;
			break;
		}
	}
	return  isPrime;
}
```
继承FNonAbandonableTask类 FNonAbandonableTask不可放弃的任务,无法调用Abandon中断任务。 
成员函数DoWork是实际的多线程工作内容



#### 需开发回收的模板任务 FAsyncTask
``` C++
DECLARE_DYNAMIC_DELEGATE_OneParam(FDDyOne,const TArray<int>&, IntList);
...
UFUNCTION(BlueprintCallable)
static void CreateCalculatePrime(UObject* WorldContextObject,const FDDyOne& DoneEvent,int MaxNum);
```

``` C++
FTimerHandle UniqueHandle;
void UUtiliyFunctionLibrary::CreateCalculatePrime(UObject* WorldContextObject,const FDDyOne& DoneEvent,int MaxNum)
{
	FAsyncTask<FCalculateTask>* MyTask = new FAsyncTask<FCalculateTask>(MaxNum);
	MyTask->StartBackgroundTask();
	
	FTimerDelegate CheckTaskDelegete = FTimerDelegate::CreateLambda([&,DoneEvent,MyTask,WorldContextObject]()
	{
		if(MyTask)
		{
			if (MyTask->IsDone())
            {
				UE_LOG(LogTemp,Log,TEXT("IsDone"))
            	DoneEvent.ExecuteIfBound(MyTask->GetTask().PrimeList);
				MyTask->EnsureCompletion();
				delete MyTask;
				FTimerManager& manager = WorldContextObject->GetWorld()->GetTimerManager();
				manager.ClearTimer(UniqueHandle);
            }
		}
		UE_LOG(LogTemp,Log,TEXT("TICK"));
	});

	WorldContextObject->GetWorld()->GetTimerManager().SetTimer(UniqueHandle,CheckTaskDelegete,1,true);
}
```

这里使用一个timer轮询task是否完成，如果完成，获得task内的数据并且释放线程内存回收。




#### 自动回收的模板任务 FAutoDeleteAsyncTask 
FAutoDeleteAsyncTaskF与AsyncTask类似，但是在DoWork结束后会自动回收，无需delete，一般使用在写文件操作中。



