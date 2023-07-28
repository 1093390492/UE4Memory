## 保存动态创建的Component至场景文件中
这里主要是因为创建了splinemesh数组作为公路的mesh载体，然而在editor下创建的  splinemeshcomponent在保存level的时候，发现组件全部丢失。

经过的检查，发现Actor内部有一个 component instance 数组，这个数组只有在编辑器下用UE操作添加组件的时才会调用，用C++或者蓝图add compoent是不会为这个创建出的组件添加到component instance数组中的。
解决方案也很简单， 用C++封装一下actor中的函数
```C++
void AActor::AddInstanceComponent(UActorComponent* Component)
{
	Component->CreationMethod = EComponentCreationMethod::Instance;
	InstanceComponents.AddUnique(Component);
}
```
给蓝图调用一下就可以了，这样创建出来的组件不仅可以保存，而且在outline中也能显示出来。
