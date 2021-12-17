## UE4 多模块的插件（自定义动画节点的补充）
之前做的自定义动画节点被分成了两个模块。  
一个是动画节点功能的实现，Runtime模块，也就是MultiModifyCurveAnimNode。  
另一个就是动画的蓝图节点，Editor模块，MultiModifyCurveAnimNodeEditor，在这个模块内实现的UMultiModifyCurveAnimGraphNode以用于在蓝图内展示蓝图节点。
  
#### 创建另一个模块
我这边是直接复制的当前模块的文件夹，修改对应的文件名和各类类名。  
最终文件结构如下  
![alt 界面](https://1093390492.github.io/Image/MultModuleInPlugin/0.png)

##### 除了各个类结构的更名之外有几点要注意
