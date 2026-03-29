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
在SkyAtmosphereCommonData.cpp里面将Mie和Ozone的系数也转换到自定义的LMS空间，数学上更加合理，大气渲染就不受workingColorSpace影响，全部在LMS空间。实际效果区别不大
```cpp
auto ConvertCoefficientsFromSRGBToLMS = [](FLinearColor CoeffSRGB)
{
    using namespace UE::Color;

    // Compute the transmittance color from the coefficients.
    FLinearColor Transmittance = FLinearColor(
        FMath::Exp(-CoeffSRGB.R),
        FMath::Exp(-CoeffSRGB.G),
        FMath::Exp(-CoeffSRGB.B));

    // Convert transmittance color from sRGB color space directly to AP1.
    Transmittance = FColorSpaceTransform(FColorSpace(EColorSpace::sRGB), FColorSpace(EColorSpace::ACESAP1),EChromaticAdaptationMethod::None).Apply(Transmittance);

    // Transform from AP1 to LMS
    FLinearColor TransmittanceLMS = FLinearColor(
        Transmittance.R * 1.106576f + Transmittance.G * -0.012113f + Transmittance.B * -0.064359f,
        Transmittance.R * -0.148505f + Transmittance.G * 1.060490f + Transmittance.B * 0.075696f,
        Transmittance.R * -0.001934f + Transmittance.G * -0.022668f + Transmittance.B * 0.949361f
    );

    // Now we have a transmittance in LMS, convert it back to coefficients.
    return FLinearColor(
        -FMath::Loge(FMath::Max(0.00001f, TransmittanceLMS.R)),
        -FMath::Loge(FMath::Max(0.00001f, TransmittanceLMS.G)),
        -FMath::Loge(FMath::Max(0.00001f, TransmittanceLMS.B)));
};
```
删掉前文Shader的colorspace调整，改为最后在AerialPerspective和SkyView或者RenderSkyAtmosphereRayMarchingPS把色彩空间转换为WorkingColorSpace。
![alt text](/assets/images/CustomRayleighColorSpace/screenshot-20260330-000055.png)
![alt text](/assets/images/CustomRayleighColorSpace/screenshot-20260330-000425.png)
**左修改后 右修改前**




[对马岛的分享]: https://advances.realtimerendering.com/s2021/jpatry_advances2021/index.html#/87/0/3
[Sébastien Hillaire]: https://sebh.github.io/publications/index.html