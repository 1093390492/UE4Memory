## UE5传递自定义Buffer数据至后处理（不编译源码）
关于在在GBuffer中写入自定义数据的功能网络上资料并不少见，大多要魔改引擎。
这里的手段主要参考 Github上一个UE的合并请求（最终打回） 
https://github.com/EpicGames/UnrealEngine/pull/8403
这里手段类似，但是做了cpp的绕过处理，直接修改了ush的指令，强制开启了CustomData的通道，所有的shadingmodel都会被额外开启两个通道，所以性能上有影响。

### $\color{Red}{警告 这里修改了引擎底层ush文件，会导致所有的项目在启动时重新编译Shader，强烈建议备份数据，包括备份引擎数据}$

#### 思考与流程
`MaterialTemplate.ush`
SceneTextureLookup函数会尝试冲场景数据中取出对应的数据，到后处理材质的节点SceneTexture返回出来。
GetScreenSpaceData(UV, false)函数则是在`DeferredShadingCommon.ush`中实现，
ScreenSpaceData数据就是从 GetScreenSpaceData(UV, false)获得。
GetScreenSpaceData -> GetGBufferData 从各种管线获取 SceneTexturesStruct数据。
获取到之后包装成FGBufferData 也就是实际包装好的GBuffer数据。

这里要注意`DeferredShadingCommon`下DecodeGBufferData包装函数里会检测是否包含CustomGBuffer数据 这里因为我们手动在usf里写入，无法通过检测。
所以需要修改函数HasCustomGBufferData 把我们想要开启CustomGbuufferdata的shadingmodelid加入函数检测列表。
```ush
bool HasCustomGBufferData(int ShadingModelID)
{
	return ShadingModelID == SHADINGMODELID_SUBSURFACE
		|| ShadingModelID == SHADINGMODELID_PREINTEGRATED_SKIN
		|| ShadingModelID == SHADINGMODELID_CLEAR_COAT
		|| ShadingModelID == SHADINGMODELID_SUBSURFACE_PROFILE
		|| ShadingModelID == SHADINGMODELID_TWOSIDED_FOLIAGE
		|| ShadingModelID == SHADINGMODELID_HAIR
		|| ShadingModelID == SHADINGMODELID_CLOTH
		|| ShadingModelID == SHADINGMODELID_EYE
	
	//------魔改-------
		|| ShadingModelID == SHADINGMODELID_DEFAULT_LIT
		|| ShadingModelID == SHADINGMODELID_UNLIT;
	//------魔改-------
}
```


---
这里打算实现将自定义数据从后处理的SceneTexture.PostProcessInput5里取出来,输出节点的实现写入`在MaterialTemplate.ush`文件里，这里包含了后处理材质里的SceneTexture节点的实现逻辑,实际上可以看到这个ush文件里包含了很多函数实现是%s的内容，而这里的%s最终会被ue替换成在编辑器内用材质编辑器编写的shadeCode。

PPI_PostProcessInput 就是这个SceneTexure里面的一个选择输出的信息 PPI_PostProcessInput0 在UE中对应的就是PostProcessInput0 本质是一个枚举Int，按照选择的SceneTexure中的输出类别，生成不同的shaderCode

修改`MaterialTemplate.ush`
在case PPI_PostProcessInput5: 把ScreenSpaceData.GBuffer.CustomData返回出来
```
...
#if POST_PROCESS_MATERIAL
		case PPI_PostProcessInput0:
			return Texture2DSample(PostProcessInput_0_Texture, bFiltered ? PostProcessInput_BilinearSampler : PostProcessInput_0_SharedSampler, UV);
		case PPI_PostProcessInput1:
			return Texture2DSample(PostProcessInput_1_Texture, bFiltered ? PostProcessInput_BilinearSampler : PostProcessInput_1_SharedSampler, UV);
		case PPI_PostProcessInput2:
			return Texture2DSample(PostProcessInput_2_Texture, bFiltered ? PostProcessInput_BilinearSampler : PostProcessInput_2_SharedSampler, UV);
		case PPI_PostProcessInput3:
			return Texture2DSample(PostProcessInput_3_Texture, bFiltered ? PostProcessInput_BilinearSampler : PostProcessInput_3_SharedSampler, UV);
		case PPI_PostProcessInput4:
			return Texture2DSample(PostProcessInput_4_Texture, bFiltered ? PostProcessInput_BilinearSampler : PostProcessInput_4_SharedSampler, UV);
		//----魔改-------
		case PPI_PostProcessInput5:
			return ScreenSpaceData.GBuffer.CustomData;
		//----魔改-------
#endif // __POST_PROCESS_COMMON__
...
```

然后在`在ShdingModelsMaterial.ush中`把上面填入的数据取出来
```ush
//添加想要填充CustomData的Shadingmodel宏
#if MATERIAL_SHADINGMODEL_UNLIT
	else if (ShadingModel == SHADINGMODELID_UNLIT)
	{
#if NUM_MATERIAL_OUTPUTS_GETCUSTOMSHADING > 0
		float4 customData =  GetCustomShading0(MaterialParameters);
		GBuffer.CustomData = customData;
#endif
	}
#endif
#if MATERIAL_SHADINGMODEL_DEFAULT_LIT
	else if (ShadingModel == SHADINGMODELID_DEFAULT_LIT)
	{
#if NUM_MATERIAL_OUTPUTS_GETCUSTOMSHADING > 0
		float4 customData =  GetCustomShading0(MaterialParameters);
		GBuffer.CustomData = customData;
#endif
	}
#endif
```
这里$\color{Red}{GetCustomShading0}$是UMaterialExpressionCustomOutput的`GetFunctionName`数据，这个名称会对应UMaterialExpressionCustomOutput扩展出来的输出节点，具体名字可以到材质里用ShaderCode模式查看。
这里给出对应代码与ShderCode
```C++

//CustomShadingModelOutput.h

// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "UObject/ObjectMacros.h"
#include "MaterialExpressionIO.h"
#include "Materials/MaterialExpressionCustomOutput.h"
#include "CustomShadingModelOutput.generated.h"

/**
 * 
 */
UCLASS()
class CUSTOMSHADINGMODEL_API UCustomShadingModelOutput : public UMaterialExpressionCustomOutput
{
	// GENERATED_BODY_LEGACY()
	GENERATED_UCLASS_BODY()
	//GENERATED_BODY()
	
	UPROPERTY(meta = (RequiredInput = "true"))
	FExpressionInput Input;

#if WITH_EDITOR
	virtual int32 Compile(class FMaterialCompiler* Compiler, int32 OutputIndex) override;
	virtual void GetCaption(TArray<FString>& OutCaptions) const override;
	virtual uint32 GetInputType(int32 InputIndex) override { return MCT_Float3; }
	virtual FExpressionInput* GetInput(int32 InputIndex) override;
#endif
	virtual int32 GetNumOutputs() const override { return 1; }
	virtual FString GetFunctionName() const override { return TEXT("GetCustomShading"); }
	virtual FString GetDisplayName() const override { return TEXT("CustomShading"); }

	
};

--------------
//CustomShadingModelOutput.cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "CustomShadingModelOutput.h"
#include "MaterialCompiler.h"

#define LOCTEXT_NAMESPACE "CustomShadingModelOutput"

UCustomShadingModelOutput::UCustomShadingModelOutput(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
{
#if WITH_EDITORONLY_DATA
	// Structure to hold one-time initialization
	struct FConstructorStatics
	{
		FText NAME_Utility;
		FConstructorStatics(const FString& DisplayName, const FString& FunctionName)
			: NAME_Utility(LOCTEXT("Utility", "Utility"))
		{
		}
	};
	static FConstructorStatics ConstructorStatics(GetDisplayName(), GetFunctionName());

	MenuCategories.Add(ConstructorStatics.NAME_Utility);

	bCollapsed = true;

	// No outputs
	Outputs.Reset();
#endif
}

#if WITH_EDITOR
int32 UCustomShadingModelOutput::Compile(FMaterialCompiler* Compiler, int32 OutputIndex)
{
	// return Super::Compile(Compiler, OutputIndex);

	if (Input.GetTracedInput().Expression)
	{
		return Compiler->CustomOutput(this, OutputIndex, Input.Compile(Compiler));
	}
	else
	{
		return CompilerError(Compiler, TEXT("Input missing"));
	}
	return INDEX_NONE;
}

void UCustomShadingModelOutput::GetCaption(TArray<FString>& OutCaptions) const
{
	// Super::GetCaption(OutCaptions);
	OutCaptions.Add(FString(TEXT("CustomShading")));
}

FExpressionInput* UCustomShadingModelOutput::GetInput(int32 InputIndex)
{
	// return Super::GetInput(InputIndex);
	return &Input;
}
#endif //

#define LOCTEXT_NAMESPACE "CustomShadingModelOutput"

```
这部分代码会在材质编辑器中生成一个自定义的输出节点
![alt 界面](https://1093390492.github.io/Image/CustomBufferData/0.jpg)
有几个注意点，这个输入类型我写的是float4，而材质编辑器中无法直接输出float4，即使是最上面的节点也只是float3，所以需要自己拼一个float4。
对应的ShaderCode
```
#define NUM_MATERIAL_OUTPUTS_GETCUSTOMSHADING 1
#define HAVE_GetCustomShading0 1
//这里对应的函数名称就是前面代码里写的GetFunctionName的返回值
MaterialFloat4 GetCustomShading0(inout FMaterialPixelParameters Parameters)
{
 return Material.PreshaderBuffer[6];
}
FLWCVector4 GetCustomShading0_LWC(inout FMaterialPixelParameters Parameters) { return LWCPromote(GetCustomShading0(Parameters)); }

```

---------
ShaderMaterialDerivedHelpers.cpp
里有个Dst.WRITES_CUSTOMDATA_TO_GBUFFER 会检查shadingmodel类型 限制了defaultlit和unlit 为false
这个数值会在后面有条件的 控制Dst.PIXELSHADEROUTPUT_MRT4的数值
这个数值会在**PixelShaderOutputCommon.ush**里控制输入输出参数
修改这个CPP需要重新编译源码 如果不想编译源码 可以强制这几个数值的开启

```
//魔改 PixelShaderOutputCommon.ush

// #if PIXELSHADEROUTPUT_MRT4
// 		, out float4 OutTarget4 : SV_Target4
// #endif
//
// #if PIXELSHADEROUTPUT_MRT5
// 		, out float4 OutTarget5 : SV_Target5
// #endif
//强制开启这两个用于输出额外数据的Pass
		, out float4 OutTarget4 : SV_Target4
		, out float4 OutTarget5 : SV_Target5
//魔改

//-------魔改----------
// #if PIXELSHADEROUTPUT_MRT4
// 	OutTarget4 = PixelShaderOut.MRT[4];
// #endif
//
// #if PIXELSHADEROUTPUT_MRT5
// 	OutTarget5 = PixelShaderOut.MRT[5];
// #endif
	OutTarget4 = PixelShaderOut.MRT[4];
	OutTarget5 = PixelShaderOut.MRT[5];
//-------魔改----------
```

理论上现在已经可以成功把数据写入我们这里定的Gbuffer内了，但是最后还有一个重点，屏幕UV和Buffer UV对不上。因为UE的ViewportUV 和 BufferUV并不是1：1对应关系，UE已经有封装好的转换函数，而其他的processpost不需要转换是因为这的processpost0~4已经被UE采样为texture，采样时的UV已经经过转换，而我们这里是直接写入的Gbuffer，需要使用BufferUV，而非ViewportUV。
![alt 界面](https://1093390492.github.io/Image/CustomBufferData/1.jpg)
