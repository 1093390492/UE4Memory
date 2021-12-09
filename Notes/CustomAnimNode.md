## UE4 自定义动画蓝图节点
项目上需要使用一连串时间单位的曲线数据来驱动播放人脸动画，控制的曲线数量有几十个，使用ModifyCuve节点一个一个的添加非常繁琐。
所以这里使用C++制作一个支持以TMap<FName,float>为入参的曲线变形动画节点。

使用ModifyCurve的节点的部分蓝图
![alt 界面](https://1093390492.github.io/Image/CustomAnimNode/0.png)
   
使用C++制作的支持TMap<FName,float>参数的变形动画节点
![alt 界面](https://1093390492.github.io/Image/CustomAnimNode/1.png)

---

首先是找到UE4内置的ModifyCurve节点的代码进行参考


```C++
//AnimNode_ModifyCurve.h
// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "CoreMinimal.h"
#include "UObject/ObjectMacros.h"
#include "Animation/AnimNodeBase.h"
#include "AnimNode_ModifyCurve.generated.h"

UENUM()
enum class EModifyCurveApplyMode : uint8
{
	/** Add new value to input curve value */
	Add,

	/** Scale input value by new value */
	Scale,

	/** Blend input with new curve value, using Alpha setting on the node */
	Blend,

	/** Blend the new curve value with the last curve value using Alpha to determine the weighting (.5 is a moving average, higher values react to new values faster, lower slower) */
	WeightedMovingAverage,

	/** Remaps the new curve value between the CurveValues entry and 1.0 (.5 in CurveValues makes 0.51 map to 0.02) */
	RemapCurve
};

/** Easy way to modify curve values on a pose */
USTRUCT(BlueprintInternalUseOnly)
struct ANIMGRAPHRUNTIME_API FAnimNode_ModifyCurve : public FAnimNode_Base
{
	GENERATED_BODY()

	UPROPERTY(EditAnywhere, EditFixedSize, BlueprintReadWrite, Category = Links)
	FPoseLink SourcePose;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, editfixedsize, Category = ModifyCurve, meta = (PinShownByDefault))
	TArray<float> CurveValues;

	UPROPERTY()
	TArray<FName> CurveNames;

	TArray<float> LastCurveValues;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = ModifyCurve, meta = (PinShownByDefault))
	float Alpha;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = ModifyCurve)
	EModifyCurveApplyMode ApplyMode;

	FAnimNode_ModifyCurve();

	// FAnimNode_Base interface
	virtual void Initialize_AnyThread(const FAnimationInitializeContext& Context) override;
	virtual void CacheBones_AnyThread(const FAnimationCacheBonesContext& Context) override;
	virtual void Evaluate_AnyThread(FPoseContext& Output) override;
	virtual void Update_AnyThread(const FAnimationUpdateContext& Context) override;
	// End of FAnimNode_Base interface

#if WITH_EDITOR
	/** Add new curve being modified */
	void AddCurve(const FName& InName, float InValue);
	/** Remove a curve from being modified */
	void RemoveCurve(int32 PoseIndex);
#endif // WITH_EDITOR
};

```

---

```C++
// Copyright Epic Games, Inc. All Rights Reserved.

#include "AnimNodes/AnimNode_ModifyCurve.h"
#include "AnimationRuntime.h"
#include "Animation/AnimInstanceProxy.h"

FAnimNode_ModifyCurve::FAnimNode_ModifyCurve()
{
	ApplyMode = EModifyCurveApplyMode::Blend;
	Alpha = 1.f;
}

void FAnimNode_ModifyCurve::Initialize_AnyThread(const FAnimationInitializeContext& Context)
{
	DECLARE_SCOPE_HIERARCHICAL_COUNTER_ANIMNODE(Initialize_AnyThread)
	Super::Initialize_AnyThread(Context);
	SourcePose.Initialize(Context);

	// Init our last values array to be the right size
	if (ApplyMode == EModifyCurveApplyMode::WeightedMovingAverage)
	{
		LastCurveValues.Reset(CurveValues.Num());
		LastCurveValues.AddZeroed(CurveValues.Num());
	}
}

void FAnimNode_ModifyCurve::CacheBones_AnyThread(const FAnimationCacheBonesContext& Context)
{
	DECLARE_SCOPE_HIERARCHICAL_COUNTER_ANIMNODE(CacheBones_AnyThread)
	Super::CacheBones_AnyThread(Context);
	SourcePose.CacheBones(Context);
}

void FAnimNode_ModifyCurve::Evaluate_AnyThread(FPoseContext& Output)
{
	DECLARE_SCOPE_HIERARCHICAL_COUNTER_ANIMNODE(Evaluate_AnyThread)
	FPoseContext SourceData(Output);
	SourcePose.Evaluate(SourceData);

	Output = SourceData;

	//	Morph target and Material parameter curves
	USkeleton* Skeleton = Output.AnimInstanceProxy->GetSkeleton();

	check(CurveNames.Num() == CurveValues.Num());

	for (int32 ModIdx = 0; ModIdx < CurveNames.Num(); ModIdx++)
	{
		FName CurveName = CurveNames[ModIdx];
		SmartName::UID_Type NameUID = Skeleton->GetUIDByName(USkeleton::AnimCurveMappingName, CurveName);
		if (NameUID != SmartName::MaxUID)
		{
			float CurveValue = CurveValues[ModIdx];
			float CurrentValue = Output.Curve.Get(NameUID);

			// Use ApplyMode enum to decide how to apply
			if (ApplyMode == EModifyCurveApplyMode::Add)
			{
				Output.Curve.Set(NameUID, CurrentValue + CurveValue);
			}
			else if (ApplyMode == EModifyCurveApplyMode::Scale)
			{
				Output.Curve.Set(NameUID, CurrentValue * CurveValue);
			}
			else if (ApplyMode == EModifyCurveApplyMode::WeightedMovingAverage)
			{
				check(LastCurveValues.Num() == CurveValues.Num());

				const float LastCurveValue = LastCurveValues[ModIdx];
				const float WAvg = FMath::WeightedMovingAverage(CurrentValue, LastCurveValue, Alpha);
				// Update the last curve value for next run
				LastCurveValues[ModIdx] = WAvg;

				Output.Curve.Set(NameUID, WAvg);
			}
			else if (ApplyMode == EModifyCurveApplyMode::RemapCurve)
			{
				const float RemapScale = 1.f / FMath::Max(1.f - CurveValue, 0.01f);
				const float RemappedValue = FMath::Min(FMath::Max(CurrentValue - CurveValue, 0.f) * RemapScale, 1.f);
				Output.Curve.Set(NameUID, RemappedValue);
			}
			else // Blend
			{
				float UseAlpha = FMath::Clamp(Alpha, 0.f, 1.f);
				Output.Curve.Set(NameUID, FMath::Lerp(CurrentValue, CurveValue, UseAlpha));
			}
		}
	}
}

void FAnimNode_ModifyCurve::Update_AnyThread(const FAnimationUpdateContext& Context)
{
	DECLARE_SCOPE_HIERARCHICAL_COUNTER_ANIMNODE(Update_AnyThread)
	// Run update on input pose nodes
	SourcePose.Update(Context);

	// Evaluate any BP logic plugged into this node
	GetEvaluateGraphExposedInputs().Execute(Context);
}

#if WITH_EDITOR

void FAnimNode_ModifyCurve::AddCurve(const FName& InName, float InValue)
{
	CurveValues.Add(InValue);
	CurveNames.Add(InName);
}

void FAnimNode_ModifyCurve::RemoveCurve(int32 PoseIndex)
{
	CurveValues.RemoveAt(PoseIndex);
	CurveNames.RemoveAt(PoseIndex);
}

#endif // WITH_EDITOR
```

代码非常清晰，想实现蓝图中的动画节点需要继承FAnimNode_Base。
因为我们的功能其实只是对原有功能的一些微调，所以必要的改动并不多。
经过分析，原来的ModifyCurve节点是通过编辑器内右键指定曲线名称增加曲线入参数量并且绑定对应的曲线数值。  
![alt 界面](https://1093390492.github.io/Image/CustomAnimNode/2.png)  
实际上就是调用AddCurve函数与RemoveCurve实现对CurveValues和CurveNames的增减，这里我们就可以公开一个TMap变量并且作为UPROPERTY暴露给蓝图，拿到这个包含了Name和Value的键值对，再到Evaluate_AnyThread函数内处理它的混合数据并且输出，而AddCurve和RemoveCurve函数可以删去了。
```C++
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = ModifyCurve, meta = (PinShownByDefault))
	TMap<FName,float> CurveNameValueMap;
```
**meta = (PinShownByDefault)**
意思是申明这个UPROPERTY在蓝图中作为一个输入的节点，这样才能将这个变量暴露出来并且可以连线赋值。

---
接下来就是改造Evaluate_AnyThread函数   
Evaluate_AnyThread函数就是实际执行的骨骼变换函数  
```C++
void FMultiModifyCurveAnimNodeStruct::Evaluate_AnyThread(FPoseContext& Output)
{
	DECLARE_SCOPE_HIERARCHICAL_COUNTER_ANIMNODE(Evaluate_AnyThread)
	FPoseContext SourceData(Output);
	SourcePose.Evaluate(SourceData);

	Output = SourceData;

	//	Morph target and Material parameter curves
	USkeleton* Skeleton = Output.AnimInstanceProxy->GetSkeleton();
	

	for(auto Elem : CurveNameValueMap)
	{
		FName CurveName = Elem.Key;
		SmartName::UID_Type NameUID = Skeleton->GetUIDByName(USkeleton::AnimCurveMappingName,CurveName);

		if(NameUID != SmartName::MaxUID)
		{
			float CurveValue = Elem.Value;
			float CurrentValue = Output.Curve.Get(NameUID);

			if(ApplyMode == EModifyCurveApplyMode_Custom::Add)
			{
				Output.Curve.Set(NameUID,CurrentValue + CurveValue);
			}
			else if(ApplyMode == EModifyCurveApplyMode_Custom::Scale)
			{
				Output.Curve.Set(NameUID,CurrentValue * CurveValue);
			}
			else if(ApplyMode == EModifyCurveApplyMode_Custom::WeightedMovingAverage)
			{
				check(LastCurveValues.Num()== CurveNameValueMap.Num())
				{
					const float LastCurveValue = LastCurveValues[Elem.Key];
					const float WAvg = FMath::WeightedMovingAverage(CurrentValue,LastCurveValue,Alpha);

					LastCurveValues[Elem.Key] = WAvg;

					Output.Curve.Set(NameUID,WAvg);
				}
			}
			else if(ApplyMode == EModifyCurveApplyMode_Custom::RemapCurve)
			{
				const float RemapScale = 1.f / FMath::Max(1.f - CurrentValue,0.01f);
				const float RemappedValue = FMath::Min(FMath::Max(CurrentValue - CurrentValue,0.f) * RemapScale,1.f);
				Output.Curve.Set(NameUID,RemappedValue);
			}
			else //Blend
			{
				float UseAlpha = FMath::Clamp(Alpha,0.f,1.f);
				Output.Curve.Set(NameUID,FMath::Lerp(CurrentValue,CurveValue,UseAlpha));
			}
		}
		
		
	}
}
```
改动并不大，实际上只是将原来的两个数组转换成TMap键值对读取而已。

这样编译出来在蓝图中是找不到我们的节点的，因为还需要一个AnimGraphNode类，这个才是动画蓝图节点的基类，他会匹配一个FAnimNode_Base在蓝图中呈现。  
依然可以参考AnimGraphNode_ModifyCurve.h  
```C++
// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "CoreMinimal.h"
#include "UObject/ObjectMacros.h"
#include "AnimGraphNode_Base.h"
#include "AnimNodes/AnimNode_ModifyCurve.h"
#include "AnimGraphNode_ModifyCurve.generated.h"

class FMenuBuilder;

/** Easy way to modify curve values on a pose */
UCLASS(MinimalAPI)
class UAnimGraphNode_ModifyCurve : public UAnimGraphNode_Base
{
	GENERATED_BODY()

	UPROPERTY(EditAnywhere, Category=Settings)
	FAnimNode_ModifyCurve Node;

public:
	UAnimGraphNode_ModifyCurve();

	// UEdGraphNode interface
	virtual FText GetTooltipText() const override;
	virtual FText GetNodeTitle(ENodeTitleType::Type TitleType) const override;
	// End of UEdGraphNode interface

	// UAnimGraphNode_Base interface
	virtual FString GetNodeCategory() const override;
	virtual void CustomizePinData(UEdGraphPin* Pin, FName SourcePropertyName, int32 ArrayIndex) const override;
	// End of UAnimGraphNode_Base interface

	// UK2Node interface
	virtual void GetNodeContextMenuActions(class UToolMenu* Menu, class UGraphNodeContextMenuContext* Context) const override;
	// End of UK2Node interface

private:
	/** Remove a curve pin with the given name */
	void RemoveCurvePin(FName CurveName);
	/** Add a curve pin with the given name */
	void AddCurvePin(FName CurveName);
	/** Create submenu with options for curves to add */
	void GetAddCurveMenuActions(FMenuBuilder& MenuBuilder) const;
	/** Create submenu with options for curves to remove */
	void GetRemoveCurveMenuActions(FMenuBuilder& MenuBuilder) const;
	/** Returns list of curves we have not added yet */
	TArray<FName> GetCurvesToAdd() const;
};
```
首先修改Node的类型为继承FAnimNode_Base的自定义动画蓝图节点类型 FMultiModifyCurveAnimNodeStruct
```C++
	UPROPERTY(EditAnywhere, Category=Settings)
	FMultiModifyCurveAnimNodeStruct Node;
```
我们这里用不到一些用户在右键菜单里动态添加Name和Value的操作，所以完全可以只重写  
GetTooltipText  
GetNodeTitle  
GetNodeCategory  
这三个函数。
```C++
FText UMultiModifyCurveUAnimGraphNode::GetTooltipText() const
{
	return GetNodeTitle(ENodeTitleType::ListView);
}

FText UMultiModifyCurveUAnimGraphNode::GetNodeTitle(ENodeTitleType::Type Title) const
{
	//return LOCTEXT("AnimGraphNode_ModifyCurve_Title", "MultiModifyCurve");
	return FText::FromString("MultiModifyCurve");
}

FString UMultiModifyCurveUAnimGraphNode::GetNodeCategory() const
{
	return TEXT("CustomAnimNode");
}
```
这样编译之后重启UE4就能在动画蓝图中使用了。

最后贴一下完整代码
```C++
/// MultiModifyCurveAnimNodeStruct.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/ObjectMacros.h"
#include "Animation/AnimNodeBase.h"
#include "MultiModifyCurveAnimNodeStruct.generated.h"


UENUM()
enum class EModifyCurveApplyMode_Custom : uint8
{
	/** Add new value to input curve value */
	Add,

	/** Scale input value by new value */
	Scale,

	/** Blend input with new curve value, using Alpha setting on the node */
	Blend,

	/** Blend the new curve value with the last curve value using Alpha to determine the weighting (.5 is a moving average, higher values react to new values faster, lower slower) */
	WeightedMovingAverage,

	/** Remaps the new curve value between the CurveValues entry and 1.0 (.5 in CurveValues makes 0.51 map to 0.02) */
	RemapCurve
};

USTRUCT(BlueprintInternalUseOnly)
struct MULTIMODIFYCURVEANIMNODE_API FMultiModifyCurveAnimNodeStruct : public FAnimNode_Base
{
	GENERATED_BODY()


	UPROPERTY(EditAnywhere, EditFixedSize, BlueprintReadWrite, Category = Links)
	FPoseLink SourcePose;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = ModifyCurve, meta = (PinShownByDefault))
	TMap<FName,float> CurveNameValueMap;

	TMap<FName,float> LastCurveValues;
	
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = ModifyCurve, meta = (PinShownByDefault))
	float Alpha;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = ModifyCurve)
	EModifyCurveApplyMode_Custom ApplyMode;

	FMultiModifyCurveAnimNodeStruct();

	// FAnimNode_Base interface
	virtual void Initialize_AnyThread(const FAnimationInitializeContext& Context) override;
	virtual void CacheBones_AnyThread(const FAnimationCacheBonesContext& Context) override;
	virtual void Evaluate_AnyThread(FPoseContext& Output) override;
	virtual void Update_AnyThread(const FAnimationUpdateContext& Context) override;
};

```

```C++
//MultiModifyCurveAnimNodeStruct.cpp
#include "MultiModifyCurveAnimNodeStruct.h"
#include "Animation/AnimInstanceProxy.h"

FMultiModifyCurveAnimNodeStruct::FMultiModifyCurveAnimNodeStruct()
{
	ApplyMode = EModifyCurveApplyMode_Custom::Blend;
	Alpha = 1.f;
}

void FMultiModifyCurveAnimNodeStruct::Initialize_AnyThread(const FAnimationInitializeContext& Context)
{
	DECLARE_SCOPE_HIERARCHICAL_COUNTER_ANIMNODE(Initialize_AnyThread)
	Super::Initialize_AnyThread(Context);
	SourcePose.Initialize(Context);

	// Init our last values array to be the right size
	if (ApplyMode == EModifyCurveApplyMode_Custom::WeightedMovingAverage)
	{

		LastCurveValues.Empty(CurveNameValueMap.Num());
		for (auto& Elem : CurveNameValueMap)
		{
			LastCurveValues.Add(Elem.Key,0);
		}
		
	}
}

void FMultiModifyCurveAnimNodeStruct::CacheBones_AnyThread(const FAnimationCacheBonesContext& Context)
{
	DECLARE_SCOPE_HIERARCHICAL_COUNTER_ANIMNODE(CacheBones_AnyThread)
	Super::CacheBones_AnyThread(Context);
	SourcePose.CacheBones(Context);
}

void FMultiModifyCurveAnimNodeStruct::Evaluate_AnyThread(FPoseContext& Output)
{
	DECLARE_SCOPE_HIERARCHICAL_COUNTER_ANIMNODE(Evaluate_AnyThread)
	FPoseContext SourceData(Output);
	SourcePose.Evaluate(SourceData);

	Output = SourceData;

	//	Morph target and Material parameter curves
	USkeleton* Skeleton = Output.AnimInstanceProxy->GetSkeleton();
	

	for(auto Elem : CurveNameValueMap)
	{
		FName CurveName = Elem.Key;
		SmartName::UID_Type NameUID = Skeleton->GetUIDByName(USkeleton::AnimCurveMappingName,CurveName);

		if(NameUID != SmartName::MaxUID)
		{
			float CurveValue = Elem.Value;
			float CurrentValue = Output.Curve.Get(NameUID);

			if(ApplyMode == EModifyCurveApplyMode_Custom::Add)
			{
				Output.Curve.Set(NameUID,CurrentValue + CurveValue);
			}
			else if(ApplyMode == EModifyCurveApplyMode_Custom::Scale)
			{
				Output.Curve.Set(NameUID,CurrentValue * CurveValue);
			}
			else if(ApplyMode == EModifyCurveApplyMode_Custom::WeightedMovingAverage)
			{
				check(LastCurveValues.Num()== CurveNameValueMap.Num())
				{
					const float LastCurveValue = LastCurveValues[Elem.Key];
					const float WAvg = FMath::WeightedMovingAverage(CurrentValue,LastCurveValue,Alpha);

					LastCurveValues[Elem.Key] = WAvg;

					Output.Curve.Set(NameUID,WAvg);
				}
			}
			else if(ApplyMode == EModifyCurveApplyMode_Custom::RemapCurve)
			{
				const float RemapScale = 1.f / FMath::Max(1.f - CurrentValue,0.01f);
				const float RemappedValue = FMath::Min(FMath::Max(CurrentValue - CurrentValue,0.f) * RemapScale,1.f);
				Output.Curve.Set(NameUID,RemappedValue);
			}
			else //Blend
			{
				float UseAlpha = FMath::Clamp(Alpha,0.f,1.f);
				Output.Curve.Set(NameUID,FMath::Lerp(CurrentValue,CurveValue,UseAlpha));
			}
		}
		
		
	}
}

void FMultiModifyCurveAnimNodeStruct::Update_AnyThread(const FAnimationUpdateContext& Context)
{
	DECLARE_SCOPE_HIERARCHICAL_COUNTER_ANIMNODE(Update_AnyThread)
	// Run update on input pose nodes
	SourcePose.Update(Context);

	// Evaluate any BP logic plugged into this node
	GetEvaluateGraphExposedInputs().Execute(Context);
}
```

```C++
//MultiModifyCurveUAnimGraphNode.h


// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "UObject/ObjectMacros.h"
#include "AnimGraphNode_Base.h"
#include "MultiModifyCurveAnimNodeStruct.h"
#include "MultiModifyCurveUAnimGraphNode.generated.h"

/**
 * 
 */
UCLASS()
class UMultiModifyCurveUAnimGraphNode : public UAnimGraphNode_Base
{
	GENERATED_BODY()
	
	UPROPERTY(EditAnywhere, Category=Settings)
	FMultiModifyCurveAnimNodeStruct Node;

	public:
	UMultiModifyCurveUAnimGraphNode();

	// UEdGraphNode interface
	virtual FText GetTooltipText() const override;
	virtual FText GetNodeTitle(ENodeTitleType::Type TitleType) const override;
	// End of UEdGraphNode interface

	// // UAnimGraphNode_Base interface
	virtual FString GetNodeCategory() const override;
	// // End of UAnimGraphNode_Base interface

};

```

```C++

//MultiModifyCurveUAnimGraphNode.cpp

// Fill out your copyright notice in the Description page of Project Settings.


#include "MultiModifyCurveUAnimGraphNode.h"

#define LOCTEXT_NAMESPACE "ModifyCurve"

UMultiModifyCurveUAnimGraphNode::UMultiModifyCurveUAnimGraphNode()
{
}

FText UMultiModifyCurveUAnimGraphNode::GetTooltipText() const
{
	return GetNodeTitle(ENodeTitleType::ListView);
}

FText UMultiModifyCurveUAnimGraphNode::GetNodeTitle(ENodeTitleType::Type Title) const
{
	//return LOCTEXT("AnimGraphNode_ModifyCurve_Title", "MultiModifyCurve");
	return FText::FromString("MultiModifyCurve");
}

FString UMultiModifyCurveUAnimGraphNode::GetNodeCategory() const
{
	return TEXT("CustomAnimNode");
}

#undef LOCTEXT_NAMESPACE

```