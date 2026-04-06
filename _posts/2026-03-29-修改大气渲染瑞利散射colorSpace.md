---
layout: post
title:  "[UE] 修改大气渲染瑞利散射colorSpace"
date:   2026-03-29 17:00:00 +0800
categories: post
image: /assets/images/CustomRayleighColorSpace/screenshot-20260329-120357.png
---



之前在[对马岛的分享]发现一个简单好用的大气渲染颜色改进方法。
用了一个自定义ColorSpace 让结果的颜色更加接近光谱渲染。看着效果不错于是打算在虚幻试试。
（最后对比跟对马岛分享的对比效果不太一样，但确实如分享说的减少了绿色的出现，紫色红色还是很奇怪，以后再看...）

![alt text](/assets/images/CustomRayleighColorSpace/screenshot-20260329-123323.png)
**左侧 修改后 ，右侧 修改前 （mie和absorption参数一致，关闭了后处理expand Gamut）**

![alt text](/assets/images/CustomRayleighColorSpace/screenshot-20260329-121356.png)
**对马岛的LMS色彩空间 参数**

<br>
## 转换矩阵
知道了对马岛用的色彩空间，还需要知道怎么从其他色彩空间转换到这个。虚幻提供了一些常用方法，如 WorkingColorSpace.ToAP1、ToXYZ等。多是将目前的workingColorSpace转换到ACES相关的。考虑到大气渲染是在workingColorSpace中进行的（SkyAtmosphereCommonData里面有转换 ConvertCoefficientsFromSRGBToWorkingColorSpace）。因此可以用AP1作为目标，这样就只需要计算对马岛的LMS colorspace转换到AP1（acescg）。

这里可以简单写个python程序做一下计算。import colour库
```python
source_to_xyz = colour.models.normalised_primary_matrix(
    prim0, white0
)
target_to_xyz = colour.models.normalised_primary_matrix(
    prim1, white1
)
# 计算XYZ→RGB逆矩阵
xyz_to_source = np.linalg.inv(source_to_xyz)
xyz_to_target = np.linalg.inv(target_to_xyz)

# A RGB → XYZ → B RGB
source_to_target = xyz_to_target @ source_to_xyz
# B RGB → XYZ → A RGB
target_to_source = xyz_to_source @ target_to_xyz
```
然后输入对应的参数(注意D65白点和ap1的D60要转换)就可以知道转换矩阵是
```cpp
float3x3 LMSToAP1 = {
    0.905355, 0.011633, 0.060449,
    0.126434, 0.94298, -0.066616,
    0.004863, 0.02254, 1.051873
};
float3x3 AP1ToLMS = {
    1.106576, -0.012113, -0.064359,
    -0.148505, 1.06049, 0.075696,
    -0.001934, -0.022668, 0.949361
};
```

## 修改大气渲染
### 系数
先在ue编辑器内把上图的Rayleigh scattering coefficents填进去，参考[Sébastien Hillaire]的论文，也就是虚幻大气计算的，可以通过这个与虚幻skyAtmosphere上系数的关联找到对应关系。
![alt text](/assets/images/CustomRayleighColorSpace/screenshot-20260329-164532.png)
**左：论文里提到的系数 右侧：虚幻编辑器的默认系数**

不需要去管这几个系数怎么来的，很好看出来引擎scattering的系数就是把论文数据的33.1作为scale了，其他数值对应做了除法。照样套到对马岛分享的系数上
![alt text](/assets/images/CustomRayleighColorSpace/screenshot-20260329-165145.png)

### 色彩空间（只改rayleigh）
UE默认是将天空大气组件输入的Coefficient当作srgb值，然后转换到workingColorSpace，在workingColorSpace做计算（Lut存储的值也是WorkingColorSpace的）。要直接使用对马岛分享的Rayleigh scattering coefficents在自定义的LMS的空间计算，需要做一些额外的修改。


#### 代码
然后要把瑞利散射的计算色彩空间做修改。

先创建一个控制台变量在skyAtmosphereRendering.cpp
```cpp
static TAutoConsoleVariable<int32> CVarSkyAtmosphereRayleighLMS(
	TEXT("r.SkyAtmosphere.RayleighLMS"), 0,
	TEXT("Enable custom LMS colorspace for Rayleigh scattering calculation."),
	ECVF_RenderThreadSafe | ECVF_Scalability);
```

再关闭SkyAtmosphereCommonData.cpp里面的转换
```cpp
// Rayleigh scattering
{
    IConsoleVariable* CVar = IConsoleManager::Get().FindConsoleVariable(TEXT("r.SkyAtmosphere.RayleighLMS"));
    int32 Value = CVar ? CVar->GetInt() : 0;
    RayleighScattering = (SkyAtmosphereComponent.RayleighScattering * SkyAtmosphereComponent.RayleighScatteringScale).GetClamped(0.0f, 1e38f);
    bool bCustomRayleighColorSpace = Value > 0;
    RayleighScattering = bCustomRayleighColorSpace
        ? RayleighScattering //如果使用自定义的瑞利散射色彩空间, 不转换到WorkingColorSpace
        : ConvertCoefficientsFromSRGBToWorkingColorSpace(RayleighScattering);

    RayleighDensityExpScale = -1.0f / SkyAtmosphereComponent.RayleighExponentialDistribution;
}
```





当然这个开关在SkyAtmosphereRendering也要用，给shader设置编译条件
```cpp
class FHighQualityMultiScatteringApprox : SHADER_PERMUTATION_BOOL("HIGHQUALITY_MULTISCATTERING_APPROX_ENABLED");
class FRayleighLMS : SHADER_PERMUTATION_BOOL("RAYLEIGH_LMS_ENABLED");//新增
```



全部用到瑞利散射的pass都要加上，如：
```cpp
using FPermutationDomain = TShaderPermutationDomain<FSampleCloudSkyAO, FFastSky, /*新增*/FRayleighLMS, FFastAerialPespective, FSecondAtmosphereLight, FRenderSky, FSampleOpaqueShadow, FSampleCloudShadow, FAtmosphereOnClouds, FMSAASampleCount>;
```

参考虚幻源码的PostProcessCombineLUTs.cpp，可以知道如何向Shader传递颜色相关的变换矩阵。然后就是在SkyAtmosphereRendering.cpp里面再写一遍，要改全部用到瑞利散射的pass，
（暂时不考虑大气Lut缓存和刷新的问题）


```cpp
if (CachedTransmittanceAndMultiScatteringLUTsVersion==0 
    || IncomingTransmittanceAndMultiScatteringLUTsVersion != CachedTransmittanceAndMultiScatteringLUTsVersion
    || CVarSkyAtmosphereStateVersioning.GetValueOnAnyThread() == false)
{
    bEvaluateTransmittanceAndMultiScatteringLUTs = true;
    Scene->SetCachedTransmittanceAndMultiScatteringLUTsVersion(IncomingTransmittanceAndMultiScatteringLUTsVersion);
}

//新增
FWorkingColorSpaceShaderParameters WorkingColorSpaceShaderParameters;
const FWorkingColorSpaceShaderParameters* InWorkingColorSpaceShaderParameters = reinterpret_cast<const FWorkingColorSpaceShaderParameters*>(GDefaultWorkingColorSpaceUniformBuffer.GetContents());
if (InWorkingColorSpaceShaderParameters)
{
    WorkingColorSpaceShaderParameters.ToXYZ = InWorkingColorSpaceShaderParameters->ToXYZ;
    WorkingColorSpaceShaderParameters.FromXYZ = InWorkingColorSpaceShaderParameters->FromXYZ;
    WorkingColorSpaceShaderParameters.ToAP1 = InWorkingColorSpaceShaderParameters->ToAP1;
    WorkingColorSpaceShaderParameters.FromAP1 = InWorkingColorSpaceShaderParameters->FromAP1;
    WorkingColorSpaceShaderParameters.ToAP0 = InWorkingColorSpaceShaderParameters->ToAP0;
    WorkingColorSpaceShaderParameters.FromAP0 = InWorkingColorSpaceShaderParameters->FromAP0;
}

```

然后就是给各个pass加上对应的参数设置，如
```cpp
// Transmittance LUT
FGlobalShaderMap* GlobalShaderMap = GetGlobalShaderMap(FeatureLevel);
if (bEvaluateTransmittanceAndMultiScatteringLUTs && CVarSkyAtmosphereTransmittanceLUT.GetValueOnRenderThread() > 0)
{
    RDG_EVENT_SCOPE(GraphBuilder, "SkyAtmosphere::TransmittanceLut");
            
    FRenderTransmittanceLutCS::FPermutationDomain PermutationVector;//修改
    PermutationVector.Set<FRayleighLMS>(bRayleighLmsEnabled); //新增
    TShaderMapRef<FRenderTransmittanceLutCS> ComputeShader(GlobalShaderMap,PermutationVector);//修改
    FRenderTransmittanceLutCS::FParameters * PassParameters = GraphBuilder.AllocParameters<FRenderTransmittanceLutCS::FParameters>();
    PassParameters->Atmosphere = Scene->GetSkyAtmosphereSceneInfo()->GetAtmosphereUniformBuffer();
    PassParameters->SkyAtmosphere = SkyInfo.GetInternalCommonParametersUniformBuffer();
    PassParameters->TransmittanceLutUAV = GraphBuilder.CreateUAV(FRDGTextureUAVDesc(TransmittanceLut, 0));
    //新增
    PassParameters->WorkingColorSpace = TUniformBufferRef<FWorkingColorSpaceShaderParameters>::CreateUniformBufferImmediate(
        WorkingColorSpaceShaderParameters,
        UniformBuffer_MultiFrame
    );
    FIntVector TextureSize = TransmittanceLut->Desc.GetSize();
    TextureSize.Z = 1;
    const FIntVector NumGroups = FIntVector::DivideAndRoundUp(TextureSize, FRenderTransmittanceLutCS::GroupSize);
    FComputeShaderUtils::AddPass(GraphBuilder, RDG_EVENT_NAME("TransmittanceLut"), PassFlag, ComputeShader, PassParameters, NumGroups);
}
```


<br>

#### Shader
要改的不多，只改SkyAtmospehere.usf。问题在需要色彩计算正确（或者说尽可能正确，因为很多计算会基于上一步计算出的Lut，Lut的数值又是把Mie和Ray合并存储的。没有很好的办法完全分离Mie和Ray），这里直接对最终结果L做色彩空间转换，不管transmittanceLut，会有点影响到Mie和Ozone的色彩。

```cpp
#ifndef FASTSKY_ENABLED 
#define FASTSKY_ENABLED 0 
#endif
//新增：
#ifndef RAYLEIGH_LMS_ENABLED 
#define RAYLEIGH_LMS_ENABLED 0 
#endif
```
增加变换矩阵
``` cpp
// Propagate alpha with (View.RenderingReflectionCaptureMask == 0.0f) guarantee
uint bPropagateAlphaNonReflection;

//新增
#if RAYLEIGH_LMS_ENABLED
static const float3x3 LMSToAP1 = {
	0.905355, 0.011633, 0.060449,
	0.126434, 0.94298, -0.066616,
	0.004863, 0.02254, 1.051873
};
static const float3x3 AP1ToLMS = {
	1.106576, -0.012113, -0.064359,
	-0.148505, 1.06049, 0.075696,
	-0.001934, -0.022668, 0.949361
};
#endif
```


```cpp
    // See slide 28 at http://www.frostbite.com/2015/08/physically-based-unified-volumetric-rendering-in-frostbite/ 
    float3 Sint			= (S        - S        * SampleTransmittance) / SafeMediumExtinction;	// integrate along the current step segment 
    float3 SintMieOnly	= (SMieOnly - SMieOnly * SampleTransmittance) / SafeMediumExtinction;
    float3 SintRayOnly	= (SRayOnly - SRayOnly * SampleTransmittance) / SafeMediumExtinction;
    L			+= Throughput * Sint;														// accumulate and also take into account the transmittance from previous steps
    LMieOnly	+= Throughput * SintMieOnly;
    LRayOnly	+= Throughput * SintRayOnly;
    Throughput	*= SampleTransmittance;
#endif

    tPrev = t;
}
#if RAYLEIGH_LMS_ENABLED
LRayOnly = mul((float3x3)WorkingColorSpace.FromAP1, mul(LMSToAP1, LRayOnly));
L = LMieOnly + LRayOnly;
#endif

```
这样改完就已经可以看到效果了，transmittanceLut的颜色变化也比较符合分享里面的对比
![alt text](/assets/images/CustomRayleighColorSpace/screenshot-20260329-220922.png)
**上图：对马岛分享的对比**

![alt text](/assets/images/CustomRayleighColorSpace/screenshot-20260329-220624.png)
**左修改后 右修改前**

![alt text](/assets/images/CustomRayleighColorSpace/screenshot-20260330-001118.png)
![alt text](/assets/images/CustomRayleighColorSpace/screenshot-20260330-001234.png)
**左修改后 右修改前**

<br>
### 色彩空间全改
在SkyAtmosphereCommonData.cpp里面将Mie和Ozone的系数也转换到自定义的LMS空间，数学上更加合理，大气渲染就不受workingColorSpace影响，全部在LMS空间（这里代码地面颜色没改）。删掉前文Shader的colorspace调整，改为最后在AerialPerspective和SkyView或者RenderSkyAtmosphereRayMarchingPS把色彩空间转换为WorkingColorSpace。实际效果区别不大。当然这样理论上也要重新选择Mie和Ozone的系数，或者完全根据美术风格决定，TODO+1
```cpp
// Copyright Epic Games, Inc. All Rights Reserved.

/*=============================================================================
	SkyAtmosphereCommonData.cpp
=============================================================================*/

#include "SkyAtmosphereCommonData.h"

#include "Components/SkyAtmosphereComponent.h"
#include "ColorManagement/ColorSpace.h"
#include "HAL/IConsoleManager.h"
#include "StateStream/SkyAtmosphereStateStream.h"

namespace
{
enum class ESkyAtmosphereColorSpaceMode : int32
{
	Working = 0,
	LMS = 1,
};

int32 GetSkyAtmosphereColorSpaceModeValue()
{
	IConsoleManager& ConsoleManager = IConsoleManager::Get();

	// Legacy fallback for existing projects.
	if (IConsoleVariable* LegacyLmsCVar = ConsoleManager.FindConsoleVariable(TEXT("r.SkyAtmosphere.RayleighLMS")))
	{
		return LegacyLmsCVar->GetInt() > 0 ? int32(ESkyAtmosphereColorSpaceMode::LMS) : int32(ESkyAtmosphereColorSpaceMode::Working);
	}

	return int32(ESkyAtmosphereColorSpaceMode::Working);
}

bool IsSkyAtmosphereLmsColorSpaceEnabled()
{
	return GetSkyAtmosphereColorSpaceModeValue() > int32(ESkyAtmosphereColorSpaceMode::Working);
}

FLinearColor ConvertCoefficientsFromSRGBToWorkingColorSpace(FLinearColor CoeffSRGB)
{
	using namespace UE::Color;

	const FColorSpace& WorkingColorSpace = FColorSpace::GetWorking();
	if (WorkingColorSpace.IsSRGB())
	{
		return CoeffSRGB;
	}

	// Compute transmittance from extinction coefficients.
	FLinearColor Transmittance = FLinearColor(
		FMath::Exp(-CoeffSRGB.R),
		FMath::Exp(-CoeffSRGB.G),
		FMath::Exp(-CoeffSRGB.B));

	Transmittance = FColorSpaceTransform::GetSRGBToWorkingColorSpace().Apply(Transmittance);

	return FLinearColor(
		-FMath::Loge(FMath::Max(0.00001f, Transmittance.R)),
		-FMath::Loge(FMath::Max(0.00001f, Transmittance.G)),
		-FMath::Loge(FMath::Max(0.00001f, Transmittance.B)));
}

const UE::Color::FColorSpace& GetLmsColorSpace()
{
	using namespace UE::Color;

	static const FColorSpace LmsColorSpace = FColorSpace(
		FVector2d(0.6501, 0.3495),
		FVector2d(0.1711, 0.7959),
		FVector2d(0.1520, 0.0218),
		FVector2d(0.3127, 0.3290));

	return LmsColorSpace;
}

FLinearColor ConvertCoefficientsFromSRGBToLMS(FLinearColor CoeffSRGB)
{
	using namespace UE::Color;

	// Compute transmittance from extinction coefficients.
	FLinearColor Transmittance = FLinearColor(
		FMath::Exp(-CoeffSRGB.R),
		FMath::Exp(-CoeffSRGB.G),
		FMath::Exp(-CoeffSRGB.B));

	static const FColorSpaceTransform SRGBToLMSTransform = FColorSpaceTransform(
		FColorSpace(EColorSpace::sRGB),
		GetLmsColorSpace(),
		EChromaticAdaptationMethod::None);

	Transmittance = SRGBToLMSTransform.Apply(Transmittance);

	return FLinearColor(
		-FMath::Loge(FMath::Max(0.00001f, Transmittance.R)),
		-FMath::Loge(FMath::Max(0.00001f, Transmittance.G)),
		-FMath::Loge(FMath::Max(0.00001f, Transmittance.B)));
}

FLinearColor ConvertCoefficientsFromSRGB(FLinearColor CoeffSRGB, bool bUseLmsColorSpace)
{
	return bUseLmsColorSpace ? ConvertCoefficientsFromSRGBToLMS(CoeffSRGB) : ConvertCoefficientsFromSRGBToWorkingColorSpace(CoeffSRGB);
}
}

//PRAGMA_DISABLE_OPTIMIZATION

const float FAtmosphereSetup::CmToSkyUnit = 0.00001f;			// Centimeters to Kilometers
const float FAtmosphereSetup::SkyUnitToCm = 1.0f / 0.00001f;	// Kilometers to Centimeters

FTentDistribution GetTentDistribution(const USkyAtmosphereComponent& SkyAtmosphereComponent) { return SkyAtmosphereComponent.OtherTentDistribution; }
FTentDistribution GetTentDistribution(const FSkyAtmosphereDynamicState& Ds)
{
	FTentDistribution D;
	D.TipAltitude = Ds.OtherTentDistributionTipAltitude;
	D.TipValue = Ds.OtherTentDistributionTipValue;
	D.Width = Ds.OtherTentDistributionWidth;
	return D;
}


void FAtmosphereSetup::ComputeAtmosphereVersion()
{
	uint32 Crc = 0;
	Crc = FCrc::MemCrc32((void*)&BottomRadiusKm,					sizeof(float),		Crc);
	Crc = FCrc::MemCrc32((void*)&TopRadiusKm,						sizeof(float),		Crc);
	Crc = FCrc::MemCrc32((void*)&MultiScatteringFactor,				sizeof(float),		Crc);
	const int32 ColorSpaceMode = GetSkyAtmosphereColorSpaceModeValue();
	Crc = FCrc::MemCrc32((void*)&ColorSpaceMode,					sizeof(int32),		Crc);
	Crc = FCrc::MemCrc32((void*)&RayleighScattering.R,				4 * sizeof(float),	Crc);
	Crc = FCrc::MemCrc32((void*)&RayleighDensityExpScale,			sizeof(float),		Crc);
	Crc = FCrc::MemCrc32((void*)&MieScattering.R,					4 * sizeof(float),	Crc);
	Crc = FCrc::MemCrc32((void*)&MieExtinction.R,					4 * sizeof(float),	Crc);
	Crc = FCrc::MemCrc32((void*)&MieAbsorption.R,					4 * sizeof(float),	Crc);
	Crc = FCrc::MemCrc32((void*)&MieDensityExpScale,				sizeof(float),		Crc);
	Crc = FCrc::MemCrc32((void*)&MiePhaseG,							sizeof(float),		Crc);
	Crc = FCrc::MemCrc32((void*)&AbsorptionExtinction.R,			4 * sizeof(float),	Crc);
	Crc = FCrc::MemCrc32((void*)&AbsorptionDensity0LayerWidth,		sizeof(float),		Crc);
	Crc = FCrc::MemCrc32((void*)&AbsorptionDensity0ConstantTerm,	sizeof(float),		Crc);
	Crc = FCrc::MemCrc32((void*)&AbsorptionDensity0LinearTerm,		sizeof(float),		Crc);
	Crc = FCrc::MemCrc32((void*)&AbsorptionDensity1ConstantTerm,	sizeof(float),		Crc);
	Crc = FCrc::MemCrc32((void*)&AbsorptionDensity1LinearTerm,		sizeof(float),		Crc);
	Crc = FCrc::MemCrc32((void*)&GroundAlbedo.R,					4 * sizeof(float),	Crc);
	TransmittanceAndMultiScatteringLUTsVersion = Crc;
}

template<typename T> void FAtmosphereSetup::InternalInit(const T& SkyAtmosphereComponent)
{
	// Convert Tent distribution to linear curve coefficients.
	auto TentToCoefficients = [](const FTentDistribution& Tent, float& LayerWidth, float& LinTerm0, float&  LinTerm1, float& ConstTerm0, float& ConstTerm1)
	{
		if (Tent.Width > 0.0f && Tent.TipValue > 0.0f)
		{
			const float px = Tent.TipAltitude;
			const float py = Tent.TipValue;
			const float slope = Tent.TipValue / Tent.Width;
			LayerWidth = px;
			LinTerm0 = slope;
			LinTerm1 = -slope;
			ConstTerm0 = py - px * LinTerm0;
			ConstTerm1 = py - px * LinTerm1;
		}
		else
		{
			LayerWidth = 0.0f;
			LinTerm0 = 0.0f;
			LinTerm1 = 0.0f;
			ConstTerm0 = 0.0f;
			ConstTerm1 = 0.0f;
		}
	};

	BottomRadiusKm = SkyAtmosphereComponent.BottomRadius;
	TopRadiusKm = SkyAtmosphereComponent.BottomRadius + FMath::Max(0.1f, SkyAtmosphereComponent.AtmosphereHeight);
	GroundAlbedo = FLinearColor(SkyAtmosphereComponent.GroundAlbedo);
	MultiScatteringFactor = FMath::Clamp(SkyAtmosphereComponent.MultiScatteringFactor, 0.0f, 100.0f);

	const bool bUseLmsColorSpace = IsSkyAtmosphereLmsColorSpaceEnabled();
	// Rayleigh scattering
	{
		RayleighScattering = (SkyAtmosphereComponent.RayleighScattering * SkyAtmosphereComponent.RayleighScatteringScale).GetClamped(0.0f, 1e38f);
		RayleighScattering = ConvertCoefficientsFromSRGB(RayleighScattering, bUseLmsColorSpace);

		RayleighDensityExpScale = -1.0f / SkyAtmosphereComponent.RayleighExponentialDistribution;
	}

	// Mie scattering
	{

		MieScattering = (SkyAtmosphereComponent.MieScattering * SkyAtmosphereComponent.MieScatteringScale).GetClamped(0.0f, 1e38f);
		MieScattering = ConvertCoefficientsFromSRGB(MieScattering, bUseLmsColorSpace);

		MieAbsorption = (SkyAtmosphereComponent.MieAbsorption * SkyAtmosphereComponent.MieAbsorptionScale).GetClamped(0.0f, 1e38f);
		MieAbsorption = ConvertCoefficientsFromSRGB(MieAbsorption, bUseLmsColorSpace);

		MieExtinction = MieScattering + MieAbsorption;
		MiePhaseG = SkyAtmosphereComponent.MieAnisotropy;
		MieDensityExpScale = -1.0f / SkyAtmosphereComponent.MieExponentialDistribution;
	}

	// Ozone
	{
		AbsorptionExtinction = (SkyAtmosphereComponent.OtherAbsorption * SkyAtmosphereComponent.OtherAbsorptionScale).GetClamped(0.0f, 1e38f);
		AbsorptionExtinction = ConvertCoefficientsFromSRGB(AbsorptionExtinction, bUseLmsColorSpace);

		TentToCoefficients(GetTentDistribution(SkyAtmosphereComponent), AbsorptionDensity0LayerWidth, AbsorptionDensity0LinearTerm, AbsorptionDensity1LinearTerm, AbsorptionDensity0ConstantTerm, AbsorptionDensity1ConstantTerm);
	}

	TransmittanceMinLightElevationAngle = SkyAtmosphereComponent.TransmittanceMinLightElevationAngle;

	UpdateTransform(SkyAtmosphereComponent.GetComponentTransform(), uint8(SkyAtmosphereComponent.TransformMode));

	ComputeAtmosphereVersion();
}


FAtmosphereSetup::FAtmosphereSetup(const USkyAtmosphereComponent& SkyAtmosphereComponent)
{
	InternalInit(SkyAtmosphereComponent);
}

FAtmosphereSetup::FAtmosphereSetup(const FSkyAtmosphereDynamicState& Ds)
{
	InternalInit(Ds);
}

void FAtmosphereSetup::ApplyWorldOffset(const FVector& InOffset)
{
	PlanetCenterKm += InOffset * double(FAtmosphereSetup::CmToSkyUnit);
}

void FAtmosphereSetup::UpdateTransform(const FTransform& ComponentTransform, uint8 TranformMode)
{
	switch (ESkyAtmosphereTransformMode(TranformMode))
	{
	case ESkyAtmosphereTransformMode::PlanetTopAtAbsoluteWorldOrigin:
		PlanetCenterKm = FVector(0.0f, 0.0f, -BottomRadiusKm);
		break;
	case ESkyAtmosphereTransformMode::PlanetTopAtComponentTransform:
		PlanetCenterKm = FVector(0.0f, 0.0f, -BottomRadiusKm) + ComponentTransform.GetTranslation() * double(FAtmosphereSetup::CmToSkyUnit);
		break;
	case ESkyAtmosphereTransformMode::PlanetCenterAtComponentTransform:
		PlanetCenterKm = ComponentTransform.GetTranslation() * double(FAtmosphereSetup::CmToSkyUnit);
		break;
	default:
		check(false);
	}
}

FLinearColor FAtmosphereSetup::GetTransmittanceAtGroundLevel(const FVector& SunDirection) const
{
	// The following code is from SkyAtmosphere.usf and has been converted to lambda functions. 
	// It compute transmittance from the origin towards a sun direction. 

	auto RayIntersectSphere = [&](FVector3f RayOrigin, FVector3f RayDirection, FVector3f SphereOrigin, float SphereRadius)
	{
		FVector3f LocalPosition = RayOrigin - SphereOrigin;
		float LocalPositionSqr = FVector3f::DotProduct(LocalPosition, LocalPosition);

		FVector3f QuadraticCoef;
		QuadraticCoef.X = FVector3f::DotProduct(RayDirection, RayDirection);
		QuadraticCoef.Y = 2.0f * FVector3f::DotProduct(RayDirection, LocalPosition);
		QuadraticCoef.Z = LocalPositionSqr - SphereRadius * SphereRadius;

		float Discriminant = QuadraticCoef.Y * QuadraticCoef.Y - 4.0f * QuadraticCoef.X * QuadraticCoef.Z;

		// Only continue if the ray intersects the sphere
		FVector2D Intersections = { -1.0f, -1.0f };
		if (Discriminant >= 0)
		{
			float SqrtDiscriminant = sqrt(Discriminant);
			Intersections.X = (-QuadraticCoef.Y - 1.0f * SqrtDiscriminant) / (2 * QuadraticCoef.X);
			Intersections.Y = (-QuadraticCoef.Y + 1.0f * SqrtDiscriminant) / (2 * QuadraticCoef.X);
		}
		return Intersections;
	};

	// Nearest intersection of ray r,mu with sphere boundary
	auto raySphereIntersectNearest = [&](FVector3f RayOrigin, FVector3f RayDirection, FVector3f SphereOrigin, float SphereRadius)
	{
		FVector2D sol = RayIntersectSphere(RayOrigin, RayDirection, SphereOrigin, SphereRadius);
		const float sol0 = sol.X;
		const float sol1 = sol.Y;
		if (sol0 < 0.0f && sol1 < 0.0f)
		{
			return -1.0f;
		}
		if (sol0 < 0.0f)
		{
			return FMath::Max(0.0f, sol1);
		}
		else if (sol1 < 0.0f)
		{
			return FMath::Max(0.0f, sol0);
		}
		return FMath::Max(0.0f, FMath::Min(sol0, sol1));
	};

	auto OpticalDepth = [&](FVector3f RayOrigin, FVector3f RayDirection)
	{
		float TMax = raySphereIntersectNearest(RayOrigin, RayDirection, FVector3f(0.0f, 0.0f, 0.0f), TopRadiusKm);

		FLinearColor OpticalDepthRGB = FLinearColor(ForceInitToZero);
		FVector3f VectorZero = FVector3f(ForceInitToZero);
		if (TMax > 0.0f)
		{
			const float SampleCount = 15.0f;
			const float SampleStep = 1.0f / SampleCount;
			const float SampleLength = SampleStep * TMax;
			for (float SampleT = 0.0f; SampleT < 1.0f; SampleT += SampleStep)
			{
				FVector3f Pos = RayOrigin + RayDirection * (TMax * SampleT);
				const float viewHeight = (FVector3f::Distance(Pos, VectorZero) - BottomRadiusKm);

				const float densityMie = FMath::Max(0.0f, FMath::Exp(MieDensityExpScale * viewHeight));
				const float densityRay = FMath::Max(0.0f, FMath::Exp(RayleighDensityExpScale * viewHeight));
				const float densityOzo = FMath::Clamp(viewHeight < AbsorptionDensity0LayerWidth ?
					AbsorptionDensity0LinearTerm * viewHeight + AbsorptionDensity0ConstantTerm :
					AbsorptionDensity1LinearTerm * viewHeight + AbsorptionDensity1ConstantTerm,
					0.0f, 1.0f);

				FLinearColor SampleExtinction = densityMie * MieExtinction + densityRay * RayleighScattering + densityOzo * AbsorptionExtinction;
				OpticalDepthRGB += SampleLength * SampleExtinction;
			}
		}

		return OpticalDepthRGB;
	};

	// Assuming camera is along Z on (0,0,earthRadius + 500m)
	const FVector3f WorldPos = FVector3f(0.0f, 0.0f, BottomRadiusKm + 0.5);
	FVector2D AzimuthElevation = FMath::GetAzimuthAndElevation(SunDirection, FVector::ForwardVector, FVector::LeftVector, FVector::UpVector); // TODO: make it work over the entire virtual planet with a local basis
	AzimuthElevation.Y = FMath::Max(FMath::DegreesToRadians(TransmittanceMinLightElevationAngle), AzimuthElevation.Y);
	const FVector3f WorldDir = FVector3f(FMath::Cos(AzimuthElevation.Y), 0.0f, FMath::Sin(AzimuthElevation.Y)); // no need to take azimuth into account as transmittance is symmetrical around zenith axis.
	FLinearColor OpticalDepthRGB = OpticalDepth(WorldPos, WorldDir);
	return FLinearColor(FMath::Exp(-OpticalDepthRGB.R), FMath::Exp(-OpticalDepthRGB.G), FMath::Exp(-OpticalDepthRGB.B));
}

void FAtmosphereSetup::ComputeViewData(
	const FVector& WorldCameraOrigin, const FVector& PreViewTranslation, const FVector3f& ViewForward, const FVector3f& ViewRight,
	FVector3f& SkyCameraTranslatedWorldOriginTranslatedWorld, FVector4f& SkyPlanetTranslatedWorldCenterAndViewHeight, FMatrix44f& SkyViewLutReferential) const
{
	// The constants below should match the one in SkyAtmosphereCommon.ush
	// Always force to be 5 meters above the ground/sea level (to always see the sky and not be under the virtual planet occluding ray tracing) and lower for small planet radius
	const float PlanetRadiusOffset = 0.005f;		

	const float Offset = PlanetRadiusOffset * SkyUnitToCm;
	const float BottomRadiusWorld = BottomRadiusKm * SkyUnitToCm;
	const FVector PlanetCenterWorld = PlanetCenterKm * SkyUnitToCm;
	const FVector PlanetCenterTranslatedWorld = PlanetCenterWorld + PreViewTranslation;
	const FVector WorldCameraOriginTranslatedWorld = WorldCameraOrigin + PreViewTranslation;
	const FVector PlanetCenterToCameraTranslatedWorld = WorldCameraOriginTranslatedWorld - PlanetCenterTranslatedWorld;
	const float DistanceToPlanetCenterTranslatedWorld = PlanetCenterToCameraTranslatedWorld.Size();

	// If the camera is below the planet surface, we snap it back onto the surface.
	// This is to make sure the sky is always visible even if the camera is inside the virtual planet.
	SkyCameraTranslatedWorldOriginTranslatedWorld = FVector3f(
						DistanceToPlanetCenterTranslatedWorld < (BottomRadiusWorld + Offset) ?
						PlanetCenterTranslatedWorld + (BottomRadiusWorld + Offset) * (PlanetCenterToCameraTranslatedWorld / DistanceToPlanetCenterTranslatedWorld) :
						WorldCameraOriginTranslatedWorld);
	SkyPlanetTranslatedWorldCenterAndViewHeight = FVector4f((FVector3f)PlanetCenterTranslatedWorld, ((FVector)SkyCameraTranslatedWorldOriginTranslatedWorld - PlanetCenterTranslatedWorld).Size());

	// Now compute the referential for the SkyView LUT
	FVector PlanetCenterToWorldCameraPos = ((FVector)SkyCameraTranslatedWorldOriginTranslatedWorld - PlanetCenterTranslatedWorld) * CmToSkyUnit;
	FVector3f Up = (FVector3f)PlanetCenterToWorldCameraPos;
	Up.Normalize();
	FVector3f Forward = ViewForward;		// This can make texel visible when the camera is rotating. Use constant world direction instead?
	//FVector3f	Left = normalize(cross(Forward, Up)); 
	FVector3f	Left;
	Left = FVector3f::CrossProduct(Forward, Up);
	Left.Normalize();
	const float DotMainDir = FMath::Abs(FVector3f::DotProduct(Up, Forward));
	if (DotMainDir > 0.999f)
	{
		// When it becomes hard to generate a referential, generate it procedurally.
		// [ Duff et al. 2017, "Building an Orthonormal Basis, Revisited" ]
		const float Sign = Up.Z >= 0.0f ? 1.0f : -1.0f;
		const float a = -1.0f / (Sign + Up.Z);
		const float b = Up.X * Up.Y * a;
		Forward = FVector3f( 1 + Sign * a * FMath::Pow(Up.X, 2.0f), Sign * b, -Sign * Up.X );
		Left = FVector3f(b,  Sign + a * FMath::Pow(Up.Y, 2.0f), -Up.Y );

		SkyViewLutReferential.SetColumn(0, Forward);
		SkyViewLutReferential.SetColumn(1, Left);
		SkyViewLutReferential.SetColumn(2, Up);
		SkyViewLutReferential = SkyViewLutReferential.GetTransposed();
	}
	else
	{
		// This is better as it should be more stable with respect to camera forward.
		Forward = FVector3f::CrossProduct(Up, Left);
		Forward.Normalize();
		SkyViewLutReferential.SetColumn(0, Forward);
		SkyViewLutReferential.SetColumn(1, Left);
		SkyViewLutReferential.SetColumn(2, Up);
		SkyViewLutReferential = SkyViewLutReferential.GetTransposed();
	}
}
```


![alt text](/assets/images/CustomRayleighColorSpace/screenshot-20260330-000055.png)
![alt text](/assets/images/CustomRayleighColorSpace/screenshot-20260330-000425.png)
**左修改后 右修改前**




[对马岛的分享]: https://advances.realtimerendering.com/s2021/jpatry_advances2021/index.html#/87/0/3
[Sébastien Hillaire]: https://sebh.github.io/publications/index.html