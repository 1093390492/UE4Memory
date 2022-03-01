## Ticker 扩充笔记 FTickerObjectBase

在编写全局事件分发器的时候，参考网络上一个现成的方案，发现分发器的管理器继承的FTickerObjectBase也实现了Tick的功能，google后并没有发现网络上有相关的内容。  
简单的扒了下代码。 没有权威资料参考，不保证完全正确。
```C++
/**
 * Base class for ticker objects
 */
class FTickerObjectBase
{
public:

	/**
	 * Constructor
	 *
	 * @param InDelay Delay until next fire; 0 means "next frame"
	 * @param Ticker the ticker to register with. Defaults to FTicker::GetCoreTicker().
	*/
#if ( PLATFORM_WINDOWS && defined(__clang__) )
	CORE_API FTickerObjectBase()							// @todo clang: non-default argument constructors needed to prevent an ICE in clang
		: FTickerObjectBase(0.0f, FTicker::GetCoreTicker())	// https://llvm.org/bugs/show_bug.cgi?id=28137
	{
	}

	CORE_API FTickerObjectBase(float InDelay)
		: FTickerObjectBase(InDelay, FTicker::GetCoreTicker())
	{
	}

	CORE_API FTickerObjectBase(float InDelay, FTicker& Ticker);
#else
	CORE_API FTickerObjectBase(float InDelay = 0.0f, FTicker& Ticker = FTicker::GetCoreTicker());
#endif

	/** Virtual destructor. */
	CORE_API virtual ~FTickerObjectBase();

	/**
	 * Pure virtual that must be overloaded by the inheriting class.
	 *
	 * @param DeltaTime	time passed since the last call.
	 * @return true if should continue ticking
	 */
	virtual bool Tick(float DeltaTime) = 0;

private:
	// noncopyable idiom
	FTickerObjectBase(const FTickerObjectBase&);
	FTickerObjectBase& operator=(const FTickerObjectBase&);

	/** Ticker to register with */
	FTicker& Ticker;
	/** Delegate for callbacks to Tick */
	FDelegateHandle TickHandle;
};

```
从名字能看出来差别 FTickerObjectBase 和 FTickableGameObject的父类FTickableObjectBase。
实际这两个的调用是不一样的逻辑。

---

###  FTickableObjectBase
FTickableObjectBase是使用的单例结构体FTickableStatics作为Tick对象的一个载体。
```C++
   struct FTickableStatics
{
	FCriticalSection TickableObjectsCritical;
	TArray<FTickableObjectBase::FTickableObjectEntry> TickableObjects;

	FCriticalSection NewTickableObjectsCritical;
	TSet<FTickableGameObject*> NewTickableObjects;

	bool bIsTickingObjects = false;

	void QueueTickableObjectForAdd(FTickableGameObject* InTickable)
	{
		FScopeLock NewTickableObjectsLock(&NewTickableObjectsCritical);
		NewTickableObjects.Add(InTickable);
	}

	void RemoveTickableObjectFromNewObjectsQueue(FTickableGameObject* InTickable)
	{
		FScopeLock NewTickableObjectsLock(&NewTickableObjectsCritical);
		NewTickableObjects.Remove(InTickable);
	}

	static FTickableStatics& Get()
	{
		static FTickableStatics Singleton;
		return Singleton;
	}
};
```
该结构体内维护一个TickableObjects,每次创建一个FTickableObjectBase类时，在构造函数内就会Get到这个单例FTickableStatics，并且把FTickableObjectBase添加入这个FTickableStatics的成员变量TickableObjects内。  
在LevelTick.cpp和GameEngine.cpp内再去调用FTickableGameObject::TickObjects这个静态方法实现Tick的完成流程。

-----------

### FTickerObjectBase
```C++
	CORE_API FTickerObjectBase(float InDelay = 0.0f, FTicker& Ticker = FTicker::GetCoreTicker());
```
FTickerObjectBase的构造函数可以看到获取的FTicker::GetCoreTicker()
```C++
FTicker& FTicker::GetCoreTicker()
{
	return TLazySingleton<FCoreTicker>::Get();
}
```
返回的就是一个FCoreTicker的单例.
```C++
FTickerObjectBase::FTickerObjectBase(float InDelay, FTicker& InTicker)
	: Ticker(InTicker)
{
	// Register delegate for ticker callback
	FTickerDelegate TickDelegate = FTickerDelegate::CreateRaw(this, &FTickerObjectBase::Tick);
	TickHandle = Ticker.AddTicker(TickDelegate, InDelay);
}
```
在FTickerObjectBase构造函数里就会把Tick函数包装成一个委托注册给这个单例FTicker.
在整个引擎代码中有多个地方调用整个单例FTicker的Tick函数，PhysScene,PreLoadScreenManager,LaunchEngineLoop...
虽然没有得到证实，整个FTicker应该是从引擎启动到关闭一直在被执行Tick的。
与FTickableStatics的Tick是被Word调用的相比，FTicker的Tick调用几乎是不间断并且调用间隔也是写死的。 从这点上来说有些类似Unity的FixedUpdate。


